# Backpressure And The Replicator

Jens Alfke — 11 November 2021

[Backpressure](https://medium.com/@jayphelps/backpressure-explained-the-flow-of-data-through-software-2350b3e77ce7) is an important consideration in the [Couchbase Lite replicator's](https://github.com/couchbase/couchbase-lite-core/blob/master/docs/overview/Replicator.md) design and implementation. It helps prevent producers from getting too far ahead of consumers and bloating memory usage.

## Back-what?

Whenever you have a multi-threaded or async producer-consumer system, the two sides can be working at different rates. If the consumer works faster than the producer, that’s fine: it’ll just be idle part of the time waiting for data. But if the _producer_ is faster, it’ll keep producing more and more data that the consumer hasn’t gotten to yet. Since that data is presumably getting buffered, the memory usage of the process keeps going up until it either starts swapping or is killed by the OS. Not good. This actually happened in some versions of Couchbase Lite 1.x and led to a few customer reports of OOM crashes.

> (If you haven’t already clicked the link above and watched the embedded video of “I Love Lucy” and the candy factory conveyer belt, do it! It’s a perfect example of this problem.)

The usual way to fix this is to allow **backpressure**. Backpressure is the ability of the consumer to say “Yo, hang on a minute!” to the producer, causing the producer to stop until the consumer catches up. 

Simple synchronous stream APIs have backpressure built in: when you call `fwrite()`, and the file’s buffer fills up, you are _forced_ to block while the OS flushes the buffer to disk. This is good for memory usage, but bad for performance, since you could otherwise be getting other work done while the kernel does I/O.

The TCP protocol supports backpressure in the form of a “window” property and some fancy algorithms, which keep one peer from sending data faster than the other peer can read it. The receiver periodically sends back an ACK packet telling the sender how much data it’s processed so far, and the sender stops sending if it’s gotten too far ahead of the peer’s latest ACK.

## Backpressure In The Replicator

The LiteCore (and Sync Gateway) replicators can be viewed as a bunch of interconnected async producer-consumer systems, running all the way from the source database to the destination database. 

On LiteCore’s pushing side, simplifying somewhat, you have:

**Source DB → `ChangesFeed` ⇉ `Pusher` ⇉ BLIP sender ⇉ WebSocket ⇉ TCP stream**

On the pulling side:

**TCP stream ⇉ WebSocket ⇉ BLIP receiver ⇉ `Puller` ⇉ `IncomingRev` ⇉ `Inserter` → Destination DB**

The “⇉” double arrows represent async connections that need to allow the right-hand component to send backpressure to the left, so it can stop sending data. That in turn might cause the left-hand component to send backpressure to _its_ left, propagating all the way back to the source if necessary.

This shows up in several different ways in the implementation.

* In the `C4Socket` and `WebSocket` interfaces, the consumer calls back to the producer to tell it when it’s finished processing some number of bytes. The producer keeps a counter of unprocessed bytes: when it writes _n_ bytes to the consumer it adds _n_ to the counter, and when the consumer tells it it’s finished _m_ bytes, it subtracts _m_. When the counter exceeds some threshold, that’s backpressure, and the producer stops writing.
* Our [BLIP](https://github.com/couchbase/couchbase-lite-core/blob/master/Networking/BLIP/docs/BLIP%20Protocol.md) protocol has a much-simplified version of TCP’s flow control for handling large messages. Every few hundred KB the receiving side will send an `ACK` frame back to the sender telling it how many bytes of the message it’s processed. If the sender finds it’s sent significantly more bytes than it’s received `ACK`s for, it pauses until it gets an `ACK`.
* The `Puller` has a pool of `IncomingRev` objects, each of which can process one incoming revision at a time. When an `IncomingRev`’s revision has been inserted into the database, the object is recycled back into the pool. When the pool is empty, that means the `IncomingRevs` and `Inserter` are at capacity, and the Puller stops handling incoming BLIP `rev` messages.
* The `Pusher` and `Puller` both have some counters that keep track of pending operations. For example, `Pusher::_changeListsInFlight` is the number of `changes` messages that have been sent to the peer that haven’t yet gotten responses. If this reaches `kMaxChangeListsInFlight` (5), the Pusher stops asking the `ChangesFeed` for more changes from the database.

This stuff unavoidably complicates the replicator. And some of the mechanisms are clumsier (and more bug-prone) than others, especially those damned `Pusher` and `Puller` counters. There are probably better ways to design this — Swift’s “[Combine](https://developer.apple.com/documentation/combine)” framework is very elegant and [explicitly supports backpressure](https://developer.apple.com/documentation/combine/processing-published-elements-with-subscribers) — but that would take a major rewrite of the replicator. 😫

## Q & A

> Q: You’ve talked about LiteCore; how is backpressure implemented in Sync Gateway’s replicator?

A: Good question. I (Jens) have not been involved in the Sync Gateway replicator implementation in years, so I don’t trust my knowledge or memory. If a SG engineer would like to to update this document, please do!

> Q: We only need backpressure because we use asynchronous APIs between producers and consumers. Why not use synchronous calls, where the writer blocks until the reader is done? That would make it unnecessary, and simplify the code too.

A: It would, but the trouble with synchronous interfaces is that since they have to stay in lock-step, the system can only run at the speed of the slowest link. It produces an effectively single-threaded implementation that can only do one thing at a time; this is a waste because we’re running on multi-core CPUs with fancy operating systems that can do file I/O, network I/O and computation at once.

It’s especially bad to have synchronous interfaces across a network. If host A sends a message to host B, blocks until it receives a reply, then sends another message … then the time between messages is increased by the round-trip time of packets between the two peers. That’s often tens of milliseconds on average, and any intermittent network hiccup will cause both sides to grind to a halt until it passes.

> Q: Does the Couchbase Lite platform code need to worry about backpressure?

A: The platform `Replicator` class doesn’t, because it’s only interacting with the replicator at a very high level, sending “start” and “stop” messages and relaying notifications.

But the platform’s WebSocket (`C4SocketFactory`) implementation does:
* When it receives data from the network it calls `c4socket_received`, and it needs to count how much data has been passed to that and not echoed by a `C4SocketFactory.completedReceive` callback. When that count grows too big it should stop reading from the network socket.
* On the flip side, when it’s given data to write by the `C4SocketFactory.write` callback, it needs to call `c4socket_completedWrite` when that data’s been written to the network. A _lack_ of those completion calls constitutes backpressure, and makes LiteCore ease off on giving the platform code more data.