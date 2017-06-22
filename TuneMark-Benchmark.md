# TuneMark
**Jens Alfke**

TuneMark is a benchmark for Couchbase Lite and LiteCore. It involves a set of real-world operations on a realistically-sized data set (6.7MB of JSON), using CRUD, iteration and querying. It’s the evolution of some code I’ve been using since 2012 for performance tuning of Couchbase Lite on Mac and iOS: I’ve used the numbers to check whether optimizations I make are working, and I’ve used the Instruments app to profile the running benchmark and look for hot spots.

There’s nothing very scientific about this set of operations and it could probably be improved; in fact we *should* improve it at first. But once we start using it to compare performance across platforms and over time, we’ll need to nail it down more, so past and present numbers are comparable. Of course we can *add* more operations to it later, and create other benchmarks too with other data sets.

## Implementations

* Objective-C: [TunesPerfTest.mm](https://github.com/couchbase/couchbase-lite-ios/blob/feature/2.0/Objective-C/Tests/TunesPerfTest.mm) in couchbase-lite-ios. Run as part of the Xcode project’s `PerfTests-Mac` and `PerfTests-iOS` schemes.

## Data Set

The data set consists of a JSON representation of an iTunes music library. It in fact derives from my (Jens Alfke’s) music library at some point in 2011 or 2012, converted from the XML format iTunes generates.
This lives in a 6.7MB text file called iTunesMusicLibrary.json, which can be found [here](https://github.com/couchbase/couchbase-lite-core/blob/master/C/tests/data/iTunesMusicLibrary.json). Each of the 12,189 lines of the file is a JSON object representing a single track; they look like this:

    {"Year":1997,"Kind":"AAC audio file","Genre":"Alternative","Name":"Syndir Guos (Opinberun Frelsarans)","Track ID":18022,"Total Time":465684,"Album":"Von","Persistent ID":"A2F441604C2B4919","Date Added":"2008-08-07T05:18:51.000Z","Track Type":"Remote","Artist":"Sigur Rós","Size":11614406,"Sample Rate":44100,"Track Number":11,"Bit Rate":256,"Date Modified":"2011-02-26T20:03:37.000Z"}

The only properties TuneMark currently uses are `Name`, `Album` and `Artist`, but we import all of them into the database just to bulk it up more.

## Procedure

The whole test below should be run 10 times, and the results of each operation averaged across runs, because the individual times are pretty variable. I use LiteCore’s `Benchmark` class to collect the times, compute averages and standard deviations, and log them.

(TODO: Define a formula to combine these numbers into one result. Just add them up? Weighted average?)

### A. Preliminaries

1. **Create DB:** Create a new empty database.
2. **Parse:** Read the JSON file line by line and parse each line into an in-memory dictionary/map object. This is not timed since it has nothing to do with Couchbase Lite.

### B. Timed Operations

Note: All operations that create or update documents should be wrapped in `inBatch` blocks so they run faster.

Note: Don’t time creating Query objects; we don’t really care about performance of that. But do time creating indexes.

1. **Import:** Iterate over the parsed JSON objects. For each one:
	1. create a new document whose ID is equal to its `Persistent ID` property  *[any objects that don’t have a `Persistent ID` should be skipped.]*
	2. store all the JSON properties into it 
	3. save it.
2. **Update Play Counts:** Iterate over all documents in the database. For each document:
	1. read the `Play Count` property as an integer (defaulting to 0), 
	2. add one, 
	3. write that back to the same property,
	4.  save the document.
3. **Update Artist Names:** Iterate over all documents in the database. For each document:
	1. If the “Artist” property begins with `The `:
		1. delete that prefix (including the space), 
		2. update the property, 
		3. save the document.
4. **Query All Artists:** 
	1. Create a query equivalent to `SELECT Artist WHERE Artist not missing and Compilation is missing GROUP BY lower(Artist) ORDER BY lower(Artist)`. *(Don’t time this.)*
	2. Run the query and collect all the artist names into an array.
	3. Optional: Verify that there are 1,115 items in the array.
	4. Save the array in a variable for later use in step 7.
5. **Index Artists:** Create an index on `(lower(Artist), Compilation)`.
6. **Query All Artists Faster:** Repeat step 4. It will be much faster this time thanks to the index, but should of course return the same results.
7. **Query Albums By Artist:**
	1. Create a query equivalent to `SELECT Album WHERE lower(Artist) = lower()$ARTIST) and Compilation is missing GROUP BY lower(Album) ORDER BY lower(Album)`. *(Don’t time this.)*
	2. Iterate over the array of artist names from step 4. For each artist:
		1. Substitute the artist name for the variable `ARTIST` in the query.
		2. Run the query, collecting each album name in an array.
		3. Add the number of albums to a running total.
	3. Optional: verify that the total is 1,887.
8. **Create Full-Text Index:** Create a full-text index on the `Name` property.
9. **Full-Text Search:** 
	1. Create a query equivalent to `SELECT Artist, Album, Name WHERE Name match ‘Rock’’ ORDER BY lower(Artist), lower(Album)`. *(Don’t time this.)*
	2. Run the query and collect the `Name` values into an array.
	3. Optional: Verify that there are 27 items in the array.