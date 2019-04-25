## April 2019 (`feature/cbl_c` branch)

_These changes are not on the master branch yet!_ They're being made to support the forthcoming CBL C binding, but are generally useful for the other bindings too.

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
