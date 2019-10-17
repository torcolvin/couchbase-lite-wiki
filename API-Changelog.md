## October 17, 2019 (`feature/xsockets` branch)

* Added `c4_runAsyncTask()`, which asynchronously calls a function on a background thread (or on a GCD queue, on Apple platforms.)

## October 9, 2019 (`feature/xsockets` branch)

* Added `c4db_startHousekeeping()`, which starts a background task that automatically purges expired documents, so the client doesn't need to call `c4db_purgeExpiredDocs()` any more.

## October 2, 2019 (`feature/xsockets` branch)

To speed up builds (especially rebuilds after touching a header) I moved declarations around to reduce the number of public headers being `#include`d during compilation. Mostly this shouldn't affect clients.

* Moved declarations of common reference types (C4Database, etc.) to `c4Base.h`
* Moved retain/release function declarations to `c4Base.h`.
* Moved `C4Address` declaration from `c4Socket.h` to `c4Replicator.h`
* Moved the C++ helper `Transaction` class from `c4.hh` to new header `c4Transaction.hh`
* Removed now-unnecessary `#include`s of other public headers at the tops of public headers.
* `c4.h` now includes all the public headers (since LiteCore itself doesn't include it anymore, it doesn't affect build times.)

## September 27, 2019 (`feature/xsockets` branch)

* Added `C4Cert` and `C4KeyPair` APIs (`c4Certificate.h`)
* Ref-counting changes:
  * Removed already-deprecated `c4db_free` and `c4doc_free`; use the preferred names `c4db_release` and `c4doc_release` instead.
  * Made `C4Database` and `C4Query` retain/release functions into inline wrappers around new generic `c4base_retain` and `c4base_release` functions.

## September 25, 2019 (`feature/xsockets` branch)

* Added TLS client cert configuration options for C4Replicator and C4Listener.

## September 17, 2019 (`feature/xsockets` branch)

Big improvements to C4Replicator, which can now take on a lot of the work that was being done by platform code. A CBL Replicator implementation can now create a single `C4Replicator` instance and use it for its entire lifespan, which should simplify the code.

* Refactored `c4repl_new()`. The existing function is now only for creating networked replicators. There's a new function `c4repl_newLocal()` for creating a local-to-local replicator.
* C4Replicators can be restarted after they stop (call `c4repl_start()` again.)
* C4Replicators can now automatically retry failed connections.
  * Added `c4repl_retry()`.
  * Added `kC4ReplicatorOptionMaxRetries` to override the number of retries.
  * Added `c4repl_setHostReachable()`, to pause/resume automatic retry attempts while offline.
* C4Replicators can now suspend when the app needs to become inactive. Added `c4repl_setSuspended()`.
* Added `C4ReplicatorStatus.flags` which indicates whether the replicator will retry, is suspended, and considers the host reachable.
* Added `c4repl_setOptions()` so replicator options can be changed in between retry attempts.
* Removed `C4ReplicatorParameters.dontStart`. Replicators now _never_ start when created; you have to call `c4repl_start()`.


## September 9, 2019 (`feature/xsockets` branch)

* Added `kC4ReplicatorOptionRootCerts`, to override the set of trusted root TLS certificates.

## August 28, 2019 (`feature/xsockets` branch)

* Added `kC4ReplicatorOptionProxyServer`, and other constants for the keys that go in that dictionary.

## August 22, 2019

* Added `C4ReplicatorParameters.dontStart`. If true, the replicator will not start until you call the new function `c4repl_start()`. This allows you to store the replicator reference before any progress callbacks are called.

## August 15, 2019 (`feature/xsockets` branch)

* Added new `C4NetworkErrorCode` constants: `kC4NetErrTLSCertRevoked`, `kC4NetErrTLSCertNameMismatch`.

## August 8, 2019 (`feature/xsockets` branch)

* Added TLS configuration options to `C4ListenerConfig`.

## July 26, 2019 (`feature/xsockets` branch)

* Added new `C4ErrorDomain` constant `MbedTLSDomain`, for mbedTLS error codes.

## May 28, 2019

* **Merged the `feature/cbl_c` branch into master!** See below for the API changes.

## May 2, 2019

* Added `c4key_setPassword`, which converts a password into an AES256 key. Please switch to using this instead of running PBKDF yourself, so we can ensure cross-platform compatibility.

## April 30, 2019

* The Fleece function `FLJSON5_ToJSON` has added two more parameters, which return an error message and the position of the error in the input text.

## April 29, 2019 (`feature/cbl_c` branch)

* Added an error parameter to `c4doc_getExpiration`.

## April 25, 2019

* Added `c4query_new2`, which takes a `language` parameter that can be JSON or N1QL. c4query_new is kept for backward compatibility but is semi-deprecated.

## March/April 2019

* `c4queryenum_seek` now allows a `rowIndex` of `-1`. This seeks back to _before_ the first row, returning the enumerator to the state it was when created.
* Added `c4queryenum_restart`, which is just a synonym for `c4queryenum_seek(e, -1)`.
* Added `c4log_willLog` which quickly tells whether a given log domain and log level will produce output. Can be tested before a call to `C4Log` if obtaining or formatting the parameters would be expensive.
* Deprecated the `kC4DB_SharedKeys` flag -- it's now ignored, because databases now always use the shared-keys optimization.

## April 2019 (`feature/cbl_c` branch)

* **Ref-counting changes:**
  - Added `c4db_release`, a better-named synonym of `c4db_free`.
  - C4Document, C4Query and C4QueryEnumerator are now ref-counted! Added `_retain` functions, and `_release` as the preferred synonym for `_free`.
* **Extra info:**
  - Added `c4db_setExtraInfo`, `c4db_getExtraInfo` which let clients associate a pointer with a C4Database, such as their Database object.
  - Similarly, added an `extraInfo` field to C4Document.
  - `C4ExtraInfo`, declared in C4Base.h, is a simple struct containing a `void*` and a "destructor" callback that (if non-NULL) will be called when the owning database/document is freed.
* **C4Database changes:**
  - Added C4Database functions that map more closely to the CBL API: `c4db_openNamed`, `c4db_copyNamed`, `c4db_deleteNamed`. These all take a parent directory and database name (without extension/suffix.)
* **C4Document changes:**
  - Added `c4doc_containingValue` which takes a `FLValue` and returns the `C4Document` it belongs to.
  - Added `c4doc_generateID` for generating document UUIDs.
  - Added `c4doc_getSingleRevision`, a more efficient version of `c4doc_get` that loads only a single revision instead of the entire tree.
* **C4Query changes:**
  - Added `c4query_setParameters`. Parameter values can now be stored with a query as in the CBL API. Pass null parameters to `c4query_run` to use the ones stored in the query.
  - Added `C4QueryObserver`. Adding an observer to a query makes it "live" as in CBL: the query will be run (in the background) with its stored parameter values, and the observer callback will be called with the enumerator when available. The query will re-run whenever the database changes, and the observer will be called every time the results change, until the observer is freed. The observer retains the query, so the client doesn't need to keep a reference to the query.
* **C4Blob changes:**
  - Added `c4stream_bytesWritten`, which returns the number of bytes written to the stream.