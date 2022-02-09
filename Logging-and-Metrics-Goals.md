This document is going to attempt to capture some guidelines and rules for implementing logging that doesn't miss the mark.  Too often we receive logs only to realize that logging is not adequate exactly in the part that we need it to be, but instead there are hundreds of other messages that provide no helpful information in the context.  Logging too much is obviously a burden too performance, so some things should be tracked as metrics and logged or retrieved later on demand.  Those will be discussed here as well:

## Logging Goals

If logging is going to be a reliable part of the debugging process going forward, then it needs to be standardized.  First let's start with the logging levels:

- ERROR - Used to indicate that something has clearly gone wrong and that incorrect results or crashing are likely imminent
- WARN - Used to indicate that something unusual has happened and has a fair chance of being an undesired outcome, but in some cases is the correct behavior
- INFO - Along the lines of "inline code comments".  Provides additional context in a given area at a high level.
- VERBOSE - Everything else.  There is no rule about this but I'd guess at least half of the logging statements in the whole library should be at this level
- DEBUG - Special case.  These are logs that only show up in debug builds.  Although given that no one runs debug builds in production, perhaps this level should be deprecated in favor of TRACE which would basically report the entry and exit of every function.

The log levels here are more or less being followed but the issue is the timing and information contained.  There needs to be an evolving list of guidelines in a system which I will incorrectly, but stubbornly refer to as LDD (Log Driven Development).  I think it's probably the case that every non trivial function in the library will need at least some logging, and that is not the case today.  There are large areas of gaps in which a lot of logic happens and makes it difficult to say more than "It went wrong between X and Y" where the gap between X and Y is dozens of lines, including function calls.  I propose the following guidelines.

- Any time a function returns early, there should be a log. 
    - Otherwise, if an investigation shows that something went wrong between X and Y, and there is a function call to Z which could return early in 6 or 7 different cases, we will have no information about which.
    - If the early return is because of an inability to proceed, it should be either an error or warning
    - If the early return is because of lack of need to proceed, it should be verbose
- There should be log statements surrounding any callout to uncontrolled code
    - This will make it easier to detect a situation where things are getting stuck in a callback to code outside of our control
- Every non-trivial (> 10 lines) function should have at least one log statement
    - When too many functions have no logging at all, it becomes difficult to narrow down where things are going wrong, especially in a hang situation
- If possible, logs should be able to print metrics automatically
    - If not possible, there should be a way to signal logs to print metrics such as messages which have not been responded to yet
- Verbose logs should include enqueuing messages to actors, and subsequently when messages are executed as part of the actor's mailbox
    - This is related to actor stack traces, but they are not a requirement for it

## Metrics

There are some various metrics that could be collected in order to identify bottlenecks, or even potentially identify deadlocks or other hangs.  This is a list of metrics that could provide debugging or other value:

- When a delegate or callback is employed, track how long is spent inside of the code outside of the control of the governing unit.  This could help identify slow performance due to callbacks outside of the code base.
- Track which locks are currently locked and by which thread, and which threads are waiting for locks (ignore areas such as the mailbox, where waiting for locks is a standard and non-contentious part of normal flow)
- Keep a manifest of recently called methods per thread (or per queue if applicable).  This can help identify the flow of a hard to diagnose bug.
- Keep a manifest of enqueue and execution points of each actor (both as a "call stack" tracing back to the original enqueue by a non-actor thread / queue and as a list of methods that a given thread or queue recently executed).  There is currently no easy way to trace the execution of an actor backwards in time.
- Keep a list of internal exceptions and events (similar to the JVM) that would be notable and the time at which they occurred.  