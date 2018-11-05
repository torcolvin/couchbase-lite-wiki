This document describes how LiteCore stores data in its underlying SQLite database. 

**You don't need to know any of this unless you work on LiteCore, or want to troubleshoot a database at a very low level.**

If you just want to inspect a database, use the [['cblite'|The 'cblite' Tool]] command-line tool. The `sqlite3` tool isn't very useful, even with the knowledge found below, because (a) most of the interesting data is encoded in binary formats, and (b) most mutating operations will fail because they invoke triggers that use custom functions not available outside LiteCore.

# KeyStores

LiteCore's low-level storage layer manages **DataFile**s, which support multiple **KeyStores**, each of which contains **Record**s. Currently LiteCore creates and uses three KeyStores:

* `default` — Documents
* `info` — Various metadata values, like the database's UUIDs
* `checkpoints` — Replicator checkpoints

A DataFile is implemented a SQLite database file, and KeyStores are tables whose names are prefixed with "`kv_`". So, `kv_default` is the table containing documents.

A KeyStore's table has the following SQL schema:
```
    (  key TEXT PRIMARY KEY,  sequence INTEGER,  flags INTEGER DEFAULT 0,  version BLOB,  body BLOB);
```

* `key` is the record's name. (In the default KeyStore this is the document-ID.)
* `sequence` is the sequence number, a number that's bumped every time the record is updated.
* `flags` stores the `C4DocumentFlags`.
* `version` stores versioning info. (Only the default KeyStore uses this, for revision IDs in a binary encoding.)
* `body` is the record's data. The storage layer doesn't interpret this data at all. (In the default KeyStore it's a revision tree encoded by the `RawRevTree` class.)

There is also a `kvmeta` table that stores the latest sequence number of each KeyStore:
```
    CREATE TABLE kvmeta (name TEXT PRIMARY KEY, lastSeq INTEGER DEFAULT 0) WITHOUT ROWID;
```

# Indexes

In the storage architecture, indexes and queries belong to a KeyStore, so it's possible for multiple KeyStores to have indexes. However, the higher-level database layer only makes use of indexes on the default (document) KeyStore, and the discussion below assumes that.

(Most databases also contain a SQLite index named `kv_default_seqs`, which is created automatically the first time a LiteCore query is ordered by sequence.)

The discussion below describes the schema of an index named "`NAME`".

## Value Indexes

A value index is simply a SQLite index named "`NAME`" on the table `kv_default`. Instead of indexing a column, it indexes an expression, which is translated to SQL from the original LiteCore JSON syntax.

## Full-Text (FTS) Indexes

A full-text index is a SQLite FTS4 virtual table named `kv_default::NAME`:
```
CREATE VIRTUAL TABLE "kv_default::NAME" USING fts4("contact.address.street", tokenize=unicodesn);
```
(SQLite FTS4 also creates some real SQL tables for internal use, which are named after the virtual table with `_content`, `_segments`, etc. appended.)

LiteCore creates some SQL triggers on `kv_default` that update the FTS4 table when a record changes. These are named after the virtual table with `::ins`, `::upd`, `::del` appended.

## Array (UNNEST) Indexes

> _Array indexes are an Iridium (post-2.1) feature_

An array index creates a SQL table named `kv_default:unnest:PATH`, where `PATH` is the property path being indexed. This table contains a row for _each individual array element_ in every document that contains an array at that path.

```
CREATE TABLE "kv_default:unnest:PATH" (docid INTEGER NOT NULL REFERENCES kv_default(rowid),
                                       i INTEGER NOT NULL, 
                                       body BLOB NOT NULL,
                                       CONSTRAINT pk PRIMARY KEY (docid, i)) WITHOUT ROWID;
```

* `docid` is a foreign key pointing to the source record (document).
* `i` is the array index where this element was found.
* `body` is the Fleece-encoded value of the array element.

LiteCore creates some SQL triggers on `kv_default` that update this table when a record changes. These are named after the index table with `::ins`, `::upd`, `::del` appended.

Last but not least, since the purpose of this table is to enable efficient array queries, it also has a regular SQL index:
```
CREATE INDEX "NAME" ON "kv_default:unnest:PATH" (fl_unnested_value(body));
```

## Predictive (ML) Indexes

> _Predictive indexes are an Iridium (post-2.1) feature_
