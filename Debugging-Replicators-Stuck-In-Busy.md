Here's how to debug a situation where the replicator gets stuck in the Busy state even though its work is done and it should go to Stopped or Idle.

## Background

The replicator consists of a tree of `Worker` objects, with `Replicator` at the top. Underneath it are a `Pusher`, a `Puller`, and a `DBWorker`. (There can be others under those, but they're not relevant here.)

Each of these recomputes its activity level (`C4ReplicatorActivityLevel`) after handling an event, and notifies its parent (if any) if the level changed. The `Replicator` keeps track of its children's activity levels and uses them when computing its own. When it sees that the `Pusher` and `Puller` are both idle, it either sets its own state to Idle (if continuous) or stops the connection (if one-shot.)

The workers consult various bits of their internal state, like sizes of queues or counts of pending replies, to determine their state. When stuck-busy problems arise, it's usually because these counts are off -- something's not being removed from a queue, or a counter got incremented but not decremented.

## Debugging

>**NOTE::** You need a LiteCore build from March 2 2018 or later (commit d897a354).

Set the following environment variables:
```
LiteCoreLog=info
LiteCoreLogSync=info
LiteCoreLogSyncBusy=info
```

Run the program and look for `Sync` logs with "`activityLevel=`" or "`pushStatus`" in them:
```
17:17:32.251796| [Sync]: {Push#151} activityLevel=busy: caughtUp=1, changeLists=0, revsInFlight=0, blobsInFlight=0, awaitingReply=81408, revsToSend=0, pendingSequences=1
17:17:32.251877| [Sync]: {DBWorker#150} activityLevel=busy: pendingResponseCount=1, eventCount=5
17:17:32.251941| [Sync]: {Repl#147} pushStatus=busy, pullStatus=stopped, dbStatus=busy, progress=5565332/5646218
17:17:32.252051| [Sync]: {Repl#147} activityLevel=busy: connectionState=2
```

These lines show that:
1. the `Pusher` is busy, and lists the internal properties it derived that from.
2. the `DBWorker` is busy.
3. the `Replicator` is notified of its children's statuses, and current progress.
4. the `Replicator` updates its own activity level to busy, and shows that its connection state is 2.

(Connection states are -1 —> kDisconnected, 0 —> kClosed, 1 —> kConnecting, 2 —> kConnected, 3 —> kClosing.)

After the replicator stops logging stuff, look backwards in the log for the last of these lines. See which subcomponent(s) are busy. Look at the internal state they're logging; you can map it to instance variable names by looking at the class's `computeActivityLevel()` method. Look at where those variables get updated and try to figure out what's out of balance. Good luck!