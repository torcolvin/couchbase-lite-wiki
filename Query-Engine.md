**This is an internal design doc. You don't need to know any of this unless you work on LiteCore's query implementation.**

LiteCore executes queries by translating their JSON form into SQL, compiling the SQL into a SQLite 'statement', and then evaluating that statement. Ta-da, that's it!

...except for the details. There are a number of complications:
* Document properties are not stored in SQL columns; there's just one blob column called `body`, and the properties are encoded inside that.
* JSON documents can contain arrays; SQL has no notion of this, nor of the `UNNEST` and 'ANY'/`EVERY` features that N1QL uses to query arrays.
* JSON (and N1QL) has a `null` value that is, confusingly, unlike SQL's `null`.

## Query Lifecycle

A `C4Query` reference is a wrapper for a C++ `SQLiteQuery` instance. The first thing the `SQLiteQuery` does is instantiate a `QueryParser` and pass it the JSON query string.

The `QueryParser` translates the JSON into Fleece to make it easy to work with, then recursively descends into the node tree. As it processes each node, it writes SQL to an output stream. (This is much like the code-generation pass of a compiler, since the JSON format is already structured much like a parse tree.)

The `SQLiteQuery` then compiles the SQL and stores the compiled statement.

To evaluate the query, a `SQLiteQueryEnumerator` steps the statement, translates each output row into a Fleece array, and writes those into an outer array. The enumerator holds on to this encoded Fleece, and hands out an iterator to its rows.

>**Note:** This has the unfortunate effect of buffering all the query results in memory at once, but it prevents the problem of leaving a SQLite statement running while changes are potentially being made (in between calls to `c4queryenum_next()`) by the same database connection -- this is explicitly disallowed and can cause garbage results.

Result columns of types other than string, number, or `missing` (SQL `null`) appear as SQLite blobs containing Fleece. The enumerator detects these blobs and handles them correctly.

## Handling Document Properties

The `QueryParser` translates a document property reference into a call to the custom SQLite function `fl_value`. So for example the JSON query operation `[".name.first"]` would translate to `fl_value(body, '.name.first')`. The implementation of `fl_body` (in SQLiteFleeceFunctions.cc) does this:

1. Interprets the first parameter, a blob, as a document body, finds the current revision, and gets a Fleece pointer to it. 
2. Uses Fleece's `Path` class to traverse the key-path given in the second parameter.
3. Returns a SQLite value corresponding to the property value. If the value is an array, dictionary or data, it's encoded into Fleece and returned as a SQL blob (with a special tag marking it as Fleece-encoded.)

## Handling Arrays

Both `UNNEST` and the `ANY`/`EVERY` operators provide a sort of nested query on an array. LiteCore has two ways to translate this to SQL, depending on whether there is a [LiteCore index on that array](Database-Schema#array-unnest-indexes).

### Un-indexed

If there's no index, LiteCore uses its `fl_each` SQL function (SQLiteFleeceEach.cc) This is a complicated thing called a _SQLite Virtual Table_. The primary use of virtual tables is to create SQLite tables that aren't implemented in the normal way (this is how FTS works), but they have a secondary use as _table-valued functions_, and that's how we use `fl_each`. A table-valued function can appear in a JOIN clause as though it were a table, and can present rows of data that are specific to each row the query is processing. So what `fl_each` does is make a Fleece array look like a SQL table, like a KeyStore table in fact. Each array element appears as a row with its own `body` column containing the Fleece value.

* An UNNEST clause is translated to a JOIN on an `fl_each` call whose parameters are (like `fl_value`) the document body and the path to the array.
* An `ANY`/`EVERY` expression is translated into a nested `SELECT` statement whose `FROM` is a similar `fl_each` call, with a test on its row count.

### Indexed

If the array is indexed, the `fl_each` call is replaced by the name of the index's table, which has the same schema as `fl_each`'s virtual table. There's an `ON` condition that matches the current document's `rowid` to the index table's `docid`.