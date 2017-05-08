The LiteCoreREST library adds a small embedded HTTP server to LiteCore, which implements **a subset** of the Couchbase Lite 1.x (and CouchDB and Cloudant and PouchDB) REST API. 

At this point (April 2017) it's intended mostly as an aid for automated testing of LiteCore; but it can be extended to the full API, which would allow it to support PhoneGap apps and even serve as a passive endpoint for the 1.x replication protocol. 

(If you want to cross-reference with the actual code, look at where the handlers are registered in [Listener.cc](https://github.com/couchbase/couchbase-lite-core/blob/master/REST/Listener.cc#L55).)

## API

| Method | Path | Parameter | Description |
|--------|------|-----------|-------------|
| GET    | /  | | Server info, like the version |
| GET    | /_all_dbs | | List of all database names |
| POST   | /_replicate | | Start a replication; parameters in JSON body. [See note below] |
| GET    | /_active_tasks | | Info on active replications |
| GET    | /_db_  | | Database doc count, current sequence, etc. |
| DELETE | /_db_       | | Deletes a database |
| PUT    | /_db_       | | Creates a database |
| POST   | /_db_       | | Creates a document with an automatically generated UUID |
| GET    | /_db_/_all_docs | | List of documents including current revID |
|        |  | ?include_docs=true | Adds body of each doc |
| GET    | /_db_/_id_  | | Returns document body |
|        | |?rev=_revID_ | Revision ID to get (optional) | 
| DELETE | /_db_/_id_  | | Deletes a document |
|        | |?rev=_revID_ | Current revision ID (required) | 
| PUT    | /_db_/_id_  | | Creates or updates a document |
|        | |?rev=_revID_ | Current revision ID (required if doc exists, unless you add a `_rev` property to the JSON body) |

#### `/_replicate`

* The only properties implemented so far are `source`, `target` and `continuous`.
* Local-to-local replication (both `source` and `target` are local db names) isn't supported yet.
