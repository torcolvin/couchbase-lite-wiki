# Replicator Lifecycle

This document serves to clarify how the `C4Replicator` class behaves as a state machine and to clarify the states and transitions allowed between them. 

![State Machine Diagram](https://user-images.githubusercontent.com/2470288/76814784-a61f3b80-67f3-11ea-97a9-5073e49ba806.png)

## State List

- Stopped: The replicator is inactive.  This state serves as both a starting state and a terminal stopped state.
- Connecting: The replicator is in the process of opening and negotiating a connection with the other side of the replicator (typically this means opening a web socket connection).  If the other side is permanently unavailble (i.e. invalid URL, not found, etc) then it will transition back to stopped and report the error.  If the other side has a timeout (and the replicator is continuous), or some other sort of network or recoverable error then the replicator will retry, and switch to offline while it is unable to connect.
- Busy: This state represents the replicator actively handling data (both inbound and outbound).  Any interruption will trigger either a switch to offline (recoverable error on a continuous replicator) or stopped (non-recoverable, or non-continuous replicator)
- Idle (continuous only): All current data has been processed and the replicator is waiting for more (continuous only, non-continuous replicators will switch to Stopping instead)
- Offline (continuous only): There is an issue affecting connectivity that is expected to resolve on its own **or** the application went into the background in platforms that support suspended background states.  Once the connectivity is restored, it will move back to busy.
- Stopping: Internal only state representing that a stop has begun

## Rapid Restart

Since there is a transitory state between other non-stopped states and stopped (i.e. stopping), behavior needs to be defined for what happens specifically during the stopping state.  As the diagram shows, the only path out of stopping is into stopped or offline.  To remedy this, the stopped state needs to have a "delayed start" mechanism that will trigger an automatic call to `start()` once the stopped or offline state is reached.  Furthermore, for technical reasons the offline and stopped states are very similar and need to have clear rules about how to behave.  The below table shows a list of states and the effect, if any, various calls have.  ❌ means the call is ignored, and "flag / no flag" refers to the _suspending_ flag being set while in that state.

| State Name | c4repl_start | c4repl_stop | c4repl_setSuspended(true) | c4repl_setSuspended(false) |
| --- | --- | --- | --- | --- |
| Stopped | Immediate Start |  ❌ | ❌ | ❌ |
| Connecting | ❌ | Move to stopping (no flag) | Move to stopping (flag) | ❌ |
| Busy | ❌ | Move to stopping (no flag) | Move to stopping (flag) | ❌ |
| Idle | ❌ | Move to stopping (no flag) | Move to stopping (flag) | ❌ |
| Offline | ❌ | Move to stopping (no flag) | ❌ | Move to connecting |
| Stopping (flag) | ❌ | Clear suspending flag | Clear resume flag | Set resume flag |
| Stopping (no flag) | Set resume flag | Clear resume flag | ❌ | ❌

If the offline or stopped state is reached, and the resume flag is set then the replicator will start again without delay.  If it is not, the replicator will remain in the offline or stopped state until some other trigger moves it.

## Database Usage

One of the tricky things about the replicator is that it operates on the same database that is floating around any number of other places in a given code base.  To give some historical context, two approaches were attempted and rejected for their faults:

1. Having Coucbase Lite manage a C4Replicator object that is single use only, and a new one must be created for each restart.  Since, prior to 2.8.0, there was no offline state in LiteCore (see the next section) this meant that the Couchbase Lite side needed to manage offline state.  This was undesirable because of the code duplication it caused between Couchbase Lite platforms.
1. Having a C4Replicator that lives for the same amount of time as its parent Couchbase Lite replicator, and immediately reopens the database so that its copy is unaffected by what goes on outside of it.  The flaw in this model is that some langauges have no deterministic destruction (Java, for example) and a user would be unable to delete the database due to the no longer used C4Replicator keeping it open until its parent was garbage collected.  

Thus, for 2.8.0 and above this method was agreed upon.  The C4Replicator will **not** open its own copy of the database, but instead its internal C++ Replicator class (which is creates on demand for start, and clears on stop) will do so.  This means that while the replicator is in any state other than stopped, the database cannot be deleted as the replicator will be keeping it open.  If the replicator is stopped, the database is free to be deleted, with the following caveat.

> :warning: If you close a C4Database that was used to create a C4Replicator, you will not be able to access the pending document IDs while the replicator is stopped.  See previous paragraph for an explanation of why.

## Notes on "Offline"

There are quite a few causes behind the Offline state:

1. Device is in "airplane mode", networking switched off
1. Out of range of WiFi and/or cell signal
1. WiFi router is available but itself has no connectivity (DSL is down, cable modem unplugged, bad IP configuration, etc.)
1. DNS can't resolve the hostname (DNS servers down, wrong DNS configuration on device or router, or host is on a private network and hostname is not public)
1. Proxy server is unreachable (DHCP misconfiguration, proxy is down, proxy is up but misconfigured, etc.)
1. Hostname is known, but that IP address is on an unreachable private network like an intranet
1. Other network issues along the route to the server (ISP problems, a backhoe has cut a backbone fiber line, AWS went down again, etc.)
1. Server itself is down
1. Sync Gateway is down

The replicator detects these by the errors they produce, like No such host, No route to host, Connection refused, Connection timeout, 502 Bad Gateway, 504 Gateway Timeout. These cause a transition to the Offline state.

**Prior to 2.8.0, the Offline state was not part of LiteCore**: it was implemented by the per-platform Couchbase Lite code, and LiteCore did not attempt to recover.  As of 2.8.0 LiteCore itself will react to errors and transition to the offline state when appropriate.  It will also enter an exponential backoff retry loop, and Couchbase Lite should make use of the `c4repl_setHostReachable` API to avoid needless retries while there is no Internet connection.  The retry timer's interval begins at 2 seconds, doubles on every transition back from Connecting to Offline (to a maximum of 600 seconds), and resets to 2 seconds on entering the Busy state. However, a one-shot replication will give up after two failed reconnect attempts.