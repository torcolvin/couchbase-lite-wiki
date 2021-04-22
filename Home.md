
## Generally Useful Pages

* [[Build And Deploy On Linux]] -- More Linux instructions (beyond what's in the [README](../blob/master/README.md#building-it).)
* [[JSON Query Schema]] -- LiteCore's query syntax
* [[REST API]] -- If you enable this unsupported listener, here's its API
* [The cblite Tool](https://github.com/couchbaselabs/cblite) -- Super useful command-line tool for inspecting databases
* [[TLS and Crypto Terminology]] -- a glossary

## Useful For QE And Support

* [[Debugging Replicators Stuck In Busy]] -- some tips for using logging to investigate a pesky source of bugs
* [[Legacy Attachment Handling]] -- backwards compatibility with the old `_attachments` when replicating with Sync Gateway
* [Replication Protocol](https://github.com/couchbase/couchbase-lite-core/blob/master/modules/docs/pages/replication-protocol.adoc) -- Specification of the WebSocket-based protocol

## For Couchbase Lite Implementors

* [C API Documentation](https://couchbase.github.io/couchbase-lite-core/C/html/modules.html) -- Doxygen-generated documentation of the C API called by Couchbase Lite
* [[API Changelog]] -- Chronological list of LiteCore C API changes
* [Using Fleece](https://github.com/couchbaselabs/fleece/wiki/Using-Fleece) and [Advanced Fleece](https://github.com/couchbaselabs/fleece/wiki/Advanced-Fleece) -- the JSON-like storage format and API used for documents
* [Shared Keys In Fleece](https://github.com/couchbaselabs/fleece/blob/master/SharedKeys.md) -- describes an often-confusing optimization of Fleece
* [Slices](https://github.com/couchbaselabs/fleece/wiki/Slices) -- a core data structure used by LiteCore and Fleece
* [[Proxy Servers]] -- LiteCore doesn't deal with them, but someone has to...

## Internal Design Documents

* [Class Overview](https://github.com/couchbase/couchbase-lite-core/blob/master/docs/overview/index.md) and [Replicator Class Overview](https://github.com/couchbase/couchbase-lite-core/blob/master/docs/overview/Replicator.md) -- A tour of the implementation's internal C++ classes
* [[Database Schema]] -- What's in the SQLite database
* [[Revision Trees]] -- Deep innards of how documents are stored
* [[Query Engine]] -- How we implemented N1QL-like queries
* [[Replication Lifecycle]] -- The state machine the replicator goes through
* [Actor School](https://github.com/couchbase/couchbase-lite-core/blob/master/Networking/BLIP/docs/Actors.md) -- The Actor concurrency library the replicator classes use
* [BLIP](https://github.com/couchbase/couchbase-lite-core/blob/master/Networking/BLIP/docs/BLIP%20Protocol.md) -- The network protocol underlying the replicator