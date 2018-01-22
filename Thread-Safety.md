## The General Rule

LiteCore has very little explicit thread-safety built into its API layer. Instead it expects you to follow the general rule:

> **A C4Database object, and objects derived from it, shall not be called concurrently.**

"Concurrently" applies to all the objects at once. For instance, calling into two C4Documents on two threads simultaneously would not be legal if they were associated with the same C4Database.

LiteCore objects don't have to be called on specific threads: you can call any object on any thread as long as you follow the rule above. So you can implement thread-safety by associating your own mutex with each C4Database, and locking the mutex while calling any object associated with that database.

This scope refers to in-memory objects, not files. It's perfectly legal to open two C4Databases on the same physical database file, and call them simultaneously (with a few exceptions detailed below.) In fact, the function `c4db_openAgain` exists to make this easy to do.

The general rule is conservative, and it is actually legal to call some of the functions concurrently. The rest of this document details this, by grouping the API into categories.

## Functions By Thread-Safety Type

### Thread-safe

These functions can be called at any time, **as long as their parameters remain valid during the call** (i.e. another thread isn't freeing the data at the same time):

* Library info
    - `c4_getBuildInfo`
    - `c4_getObjectCount`
    - `c4_dumpInstances`
* C4Error API
    - `c4error_make`
    - `c4error_getMessageC`
    - `c4error_mayBeTransient`
    - `c4error_mayBeNetworkDependent`
* C4Slice API
    - `c4SliceEqual`
    - `c4Slice_free`
* C4Log API
    - `c4log_`\* (entire API)
* Databases
    - `c4db_open`
    - `c4db_copy`
    - `c4db_retain`
    - `c4db_free` (as long as it was balanced by a prior `c4db_retain`; it's really "release")
* Documents
    - `c4raw_free`
    - `c4rev_getGeneration`
    - `c4doc_isOldMetaProperty`
    - `c4doc_hasOldMetaProperties`
    - `c4doc_encodeStrippingOldMetaProperties`
* Database observers
    - `c4dbobs_getChanges`
    - `c4dbobs_free`
    - `c4docobs_free`
* Replication
    - `c4repl_isValidDatabaseName`
    - `c4repl_parseURL`
    - `c4repl_getStatus`
    - `c4repl_getResponseHeaders`
    - `c4repl_stop`
    - `c4repl_free`

### Database file-exclusive

These functions can only be called if no other C4Database objects exist on the same file. Otherwise they'll return an error (not crash!)

- `c4db_delete`
- `c4db_deleteAtPath`
- `c4db_rekey`

### Database-exclusive

These functions follow the general rule: only one thread at a time can call any of these functions with a specific C4Database reference.

* Database operations
    - `c4db_openAgain`
    - `c4db_copy`
    - `c4db_close`
    - `c4db_compact`
* Database properties
    - `c4db_getPath`
    - `c4db_getConfig`
    - `c4db_getDocumentCount`
    - `c4db_getLastSequence`
    - `c4db_nextDocExpiration`
    - `c4db_getMaxRevTreeDepth`
    - `c4db_setMaxRevTreeDepth`
    - `c4db_getUUIDs`
    - `c4db_sharedFleeceEncoder` _(the code that uses the encoder needs to be database-exclusive too)_
* Transactions (see note below)
    - `c4db_beginTransaction`
    - `c4db_endTransaction`
    - `c4db_isInTransaction`
* Document I/O
    - `c4raw_get`
    - `c4raw_put`
    - `c4doc_get`
    - `c4doc_getBySequence`
    - `c4doc_put`
    - `c4doc_create`
    - `c4doc_update` (Also document-exclusive)
    - `c4doc_save` (Also document-exclusive)
    - `c4doc_setExpiration`
    - `c4doc_getExpiration`
    - `c4db_purgeDoc`
* Shared keys (Because FLSharedKeys isn't thread-safe)
    - `c4db_getFLSharedKeys`
    - `c4db_createFleeceEncoder`
    - `c4db_encodeJSON`
    - `c4db_initFLDictKey`
    - `c4doc_bodyAsJSON`
    - `c4doc_dictIsBlob`
    - `c4doc_dictContainsBlobs`
* Document expiration
    - `c4db_enumerateExpired`
    - `c4exp_next`
    - `c4exp_getDocID`
    - `c4exp_purgeExpired`
    - `c4exp_close`
    - `c4exp_free`
* Database observers
    - `c4dbobs_create`
    - `c4docobs_create`
* Queries
    - `c4query_new`
    - `c4query_free`
    - `c4query_explain`
    - `c4query_run`
    - `c4query_fullTextMatched`
    - `c4query_free`
    - `c4queryenum_next`
    - `c4queryenum_fullTextMatched`
    - `c4queryenum_getRowCount`
    - `c4queryenum_seek`
    - `c4queryenum_refresh`
    - `c4queryenum_close`
* Replication
    - `c4repl_new`
    - `c4repl_newWithSocket`
    - `c4db_getCookies`
    - `c4db_setCookie`
    - `c4db_clearCookies`
    - `c4socket_registerFactory` (Can only be called _once_)


### Transactions

Transactions have their own concurrency behavior:

> **A transaction is associated with a C4Database instance, _not_ with a thread.**

If two threads call `c4db_beginTransaction` one after the other on the same database, this is legal but creates a _single_ transaction. The second call does _not_ block until the first thread ends its transaction; instead it just increments the transaction's ref-count. Any writes performed by either C4Database instance will go through, and be committed when the second call to `c4db_endTransaction` is made.

A related effect is that there is no read isolation between threads. If one thread opens a transaction and makes changes, another thread reading via the same C4Database instance will see those changes, even before the transaction is committed.

The moral of the story is that if your threads need their own transactions, they should open separate connections. Alternatively, you could create your own per-database transaction mutex that a thread acquires before beginning a transaction and releases after ending it.

### Document-exclusive

The functions below operate on a C4Document instance in memory without accessing the database; these can be called on different documents of the same database at once, as long as each C4Document isn't called concurrently.

Direct access to C4Document's public fields obviously has the same restrictions, although concurrent reads are OK.

- `c4doc_selectRevision`  (see note)
- `c4doc_selectCurrentRevision`
- `c4doc_loadRevisionBody` (see note)
- `c4doc_detachRevisionBody`
- `c4doc_hasRevisionBody`
- `c4doc_selectParentRevision`
- `c4doc_selectNextRevision`
- `c4doc_selectNextLeafRevision`  (see note)
- `c4doc_selectFirstPossibleAncestorOf`
- `c4doc_selectNextPossibleAncestorOf`
- `c4doc_selectCommonAncestorRevision`
- `c4doc_removeRevisionBody`
- `c4doc_purgeRevision`
- `c4doc_resolveConflict`
- `c4doc_save` (Also database-exclusive)
- `c4doc_update` (Also database-exclusive)

Note: The API allows for non-current document revisions to be stored separately in the database, which would make `c4doc_loadRevisionBody` database-exclusive since it might call into the database. The same goes for `c4doc_selectRevision` and `c4doc_selectNextLeafRevision`, when the `withBody` parameter is true. _However_, the current implementation never stores revision bodies externally, so in practice these functions are not database-exclusive.

### Query-exclusive

These functions only access a C4Query's data, not the database, so their only restriction is that they can't be called on the same C4Query instance at the same time.

- `c4query_columnCount`
- `c4query_nameOfColumn`

### Socket-exclusive

The functions that pass network events to a custom C4Socket should not be called concurrently on the same C4Socket.

- `c4socket_gotHTTPResponse`
- `c4socket_opened`
- `c4socket_closeRequested`
- `c4socket_closed`
- `c4socket_completedWrite`
- `c4socket_received`
