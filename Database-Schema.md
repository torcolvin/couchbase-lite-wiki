This document describes how LiteCore stores data in its underlying SQLite database. 

**You don't need to know any of this unless you work on LiteCore, or want to troubleshoot a database at a very low level.** If you just want to inspect a database, use the [['cblite'|The 'cblite' Tool]] command-line tool. The `sqlite3` tool isn't very useful, even with the knowledge found below, because (a) most of the interesting data is encoded in binary formats, and (b) most mutating operations will fail because they invoke triggers that use custom functions not available outside LiteCore.

## 1. KeyStores

LiteCore's low-level storage layer manages **DataFile**s, which support multiple **KeyStores**, each of which contains **Record**s. Currently LiteCore creates and uses three KeyStores:

* `default` — Documents
* `info` — Various metadata values, like the database's UUIDs
* `checkpoints` — Replicator checkpoints

A DataFile is implemented a SQLite database file, and KeyStores are tables whose names are prefixed with "`kv_`". So, `kv_default` is the table containing documents.

A KeyStore's table has the following SQL schema:
```
CREATE TABLE kv_NAME (
    key TEXT PRIMARY KEY,
    sequence INTEGER,
    flags INTEGER DEFAULT 0,
    version BLOB,
    body BLOB );
```

* `key` is the record's name. (In the default KeyStore this is the document-ID.)
* `sequence` is the sequence number, a number that's bumped every time the record is updated.
* `flags` stores the `C4DocumentFlags`.
* `version` stores versioning info. (Only the default KeyStore uses this, for revision IDs in a binary encoding.)
* `body` is the record's data. The storage layer doesn't interpret this data at all. (In the default KeyStore it's a [[revision tree|Revision Trees]].)

### Sequence Support

There is also a `kvmeta` table that just stores the latest sequence number of each KeyStore:
```
CREATE TABLE kvmeta (
    name TEXT PRIMARY KEY,
    lastSeq INTEGER DEFAULT 0 )  WITHOUT ROWID;
```

Most databases also contain a SQLite index named `kv_default_seqs`, which is created automatically the first time the KeyStore is iterated in sequence order (as during a push replication.)


### Expiration

> _Expiration is a post-2.1 feature_

The first time any record in a KeyStore is given an expiration time (TTL), a new column `expiration` is added to its KeyStore's table to record it. This column contains a number (seconds since Unix epoch) in records that expire, and is null otherwise. An index `kv_default_expiration` is also created to allow efficient search of expired records.

## 2. Indexes In LiteCore 2.0/2.1

In the storage architecture, indexes and queries belong to a KeyStore (not directly to a DataFile), so it's _possible_ for multiple KeyStores to have indexes. However, the higher-level database layer only makes use of indexes on the default (document) KeyStore, and the discussion below assumes that.

The discussion below describes the schema of an index named "`NAME`".

### Value Indexes

A value index is simply a SQLite index named "`NAME`" on the table `kv_default`. Instead of indexing a column, it indexes an expression, which is translated to SQL from the original LiteCore JSON syntax.

### Full-Text (FTS) Indexes

A full-text index is a SQLite FTS4 virtual table named `kv_default::NAME`:
```
CREATE VIRTUAL TABLE "kv_default::NAME" USING fts4("contact.address.street", tokenize=unicodesn);
```
(SQLite FTS4 also creates some real SQL tables for internal use, which are named after the virtual table with `_content`, `_segments`, etc. appended.)

LiteCore creates some SQL triggers on `kv_default` that update the FTS4 table when a record changes. These are named after the virtual table with `::ins`, `::upd`, `::del` appended.

## 3. Indexes In LiteCore 'Iridium' (2.5?)

Iridium adds array (UNNEST) and predictive indexes. With the proliferation of index types, we've added a table to keep track of indexes. It's named `indexes` and has a row for each index; its columns are:

* `name` (string, primary key) is the index name as registered through the API.
* `type` (integer) is the index type, corresponding to the enums `KeyStore::IndexType` and `C4IndexType`.
* `keyStore` (string) is the name of the KeyStore being indexed (currently always `kv_default`.)
* `expression` (string) is the JSON query expression describing the index.
* `indexTableName` (string) is the name of the SQLite table created for the index.

Value indexes and full-text indexes are as described above.

### Array (UNNEST) Indexes

An array index creates a SQL table whose name begins `kv_default:unnest:`. This table contains a row for _each individual array element_ in every document that contains an array at that path.

```
CREATE TABLE "kv_default:unnest:PATH" (
    docid INTEGER NOT NULL REFERENCES kv_default(rowid),
    i INTEGER NOT NULL, 
    body BLOB NOT NULL,
    CONSTRAINT pk PRIMARY KEY (docid, i) )  WITHOUT ROWID;
```

* `docid` is a foreign key pointing to the source record (document).
* `i` is the array index where this element was found.
* `body` is the Fleece-encoded value of the array element.

LiteCore creates some SQL triggers on `kv_default` that update this table when a record changes. These are named after the index table with `::ins`, `::upd`, `::del` appended.

Last but not least, since the purpose of this table is to enable efficient array queries, it also has a regular SQL index:
```
CREATE INDEX "NAME" ON "kv_default:unnest:PATH" (fl_unnested_value(body));
```
or
```
CREATE INDEX "NAME" ON "kv_default:unnest:PATH" (fl_unnested_value(body, 'SUB_PROPERTY'));
```

(If there are multiple LiteCore indexes on the same path, but indexing different sub-properties, they share the same index table but of course create separate SQL indexes.)

### Predictive (ML) Indexes

A predictive index is much like an array index. Its name begins with `kv_default:predict:`.
```
CREATE TABLE "kv_default:predict:XXX" (
    docid INTEGER PRIMARY KEY REFERENCES kv_default(rowid),
    body BLOB NOT NULL ON CONFLICT IGNORE )  WITHOUT ROWID;
```

* `docid` is a foreign key pointing to the source record (document).
* `body` is the Fleece-encoded result of the prediction function.

LiteCore creates some SQL triggers on `kv_default` that update this table when a record changes. These are named after the predictive table with `::ins`, `::upd`, `::del` appended.

Lastly, there is a SQLite index on the predictive table, that indexes the desired result property:

```
CREATE INDEX "NAME" ON "kv_default:predict:DIGEST" (fl_unnested_value(body, 'RESULT_PROPERTY'));
```

(If there are multiple LiteCore indexes on the same prediction, but indexing different result properties, they share the same predictive table but of course create separate SQL indexes.)