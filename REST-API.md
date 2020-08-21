The LiteCoreREST library adds a small embedded HTTP server to LiteCore, which implements **a subset** of the Couchbase Lite 1.x (and CouchDB and Cloudant and PouchDB) REST API. You can easily run this by using the [cblite][CBLITE] tool's `serve` subcommand.

Again, this is not the full REST API; it doesn't expose all functionality, it's not enough for Cordova/PhoneGap, and it's not enough for compatibility with the 1.x (or CouchDB or PouchDB) replicator. But it has some uses:

* Automated testing of LiteCore
* Load testing of Sync Gateway (by starting a bunch of `cblite serve` processes to replicate with it)
* Automated creation of Couchbase Lite 2 database files from a server, to be bundled into apps

(If you want to cross-reference with the actual code, look at where the handlers are registered in [Listener.cc](https://github.com/couchbase/couchbase-lite-core/blob/master/REST/RESTListener.cc#L76).)

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
| POST   | /_db_/_bulk_docs | | Creates/updates/deletes multiple documents |
| GET    | /_db_/_id_  | | Returns document body |
|        | |?rev=_revID_ | Revision ID to get (optional) | 
| DELETE | /_db_/_id_  | | Deletes a document |
|        | |?rev=_revID_ | Current revision ID (required) | 
| PUT    | /_db_/_id_  | | Creates or updates a document |
|        | |?rev=_revID_ | Current revision ID (required if doc exists, unless you add a `_rev` property to the JSON body) |

## Missing Stuff

* All HTTP endpoints and "`?`" options not listed above like `_changes`, `_bulk_get`, ...
* Access to revision history
* Attachments
* `/_replicate` properties other than `source`, `target`, `continuous`, and `cancel`
* Local-to-local replication (where both `source` and `target` are local db names)
* Queries (obviously there are no views or design docs, but eventually it'd be nice to be able to POST a query in the JSON syntax or N1QL.)

[CBLITE]: https://github.com/couchbaselabs/couchbase-mobile-tools/blob/master/README.cblite.md