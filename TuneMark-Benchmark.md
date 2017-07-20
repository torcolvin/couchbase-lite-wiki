# TuneMark
**Jens Alfke**

TuneMark is a benchmark for Couchbase Lite and LiteCore. It involves a set of real-world operations on a realistically-sized data set (6.7MB of JSON), using CRUD, iteration and querying. It’s the evolution of some code I’ve been using since 2012 for performance tuning of Couchbase Lite on Mac and iOS: I’ve used the numbers to check whether optimizations I make are working, and I’ve used the Instruments app to profile the running benchmark and look for hot spots.

There’s nothing very scientific about this set of operations and it could probably be improved; in fact we *should* improve it at first. But once we start using it to compare performance across platforms and over time, we’ll need to nail it down more, so past and present numbers are comparable. Of course we can *add* more operations to it later, and create other benchmarks too with other data sets.

## Implementations

* Objective-C: [TunesPerfTest.mm](https://github.com/couchbase/couchbase-lite-ios/blob/feature/2.0/Objective-C/Tests/TunesPerfTest.mm) in couchbase-lite-ios. Run as part of the Xcode project’s `PerfTests-Mac` and `PerfTests-iOS` schemes.

## Data Set

The data set consists of a JSON representation of an iTunes music library. It in fact derives from my (Jens Alfke’s) music library at some point in 2011 or 2012, converted from the XML format iTunes generates.
This lives in a 6.7MB text file called iTunesMusicLibrary.json, which can be found [here](https://github.com/couchbase/couchbase-lite-core/blob/master/C/tests/data/iTunesMusicLibrary.json). Each of the 12,189 lines of the file is a JSON object representing a single track; they look like this:

`{"Year":1997,"Kind":"AAC audio file","Genre":"Alternative","Name":"Syndir Guos (Opinberun Frelsarans)","Track ID":18022,"Total Time":465684,"Album":"Von","Persistent ID":"A2F441604C2B4919","Date Added":"2008-08-07T05:18:51.000Z","Track Type":"Remote","Artist":"Sigur Rós","Size":11614406,"Sample Rate":44100,"Track Number":11,"Bit Rate":256,"Date Modified":"2011-02-26T20:03:37.000Z"}`

The only properties TuneMark currently uses are `Name`, `Album` and `Artist`, but we import all of them into the database just to bulk it up more.

## Example Results

### iOS

(iPhone 6s+, iOS 10.3.2; June 21, 2017)
```
Import 12189 docs:  Range:   2.534 ...   2.611 sec, Average:   2.560, median:   2.553, std dev: 0.0173
                    Range: 207.859 ... 214.202 us/doc, Average: 209.986, median: 209.454, std dev:  1.42
Update 1223 docs:   Range: 914.047 ... 2265.080 ms, Average: 946.194, median: 948.793, std dev:  15.6
                    Range: 747.381 ... 1852.069 us/update, Average: 773.666, median: 775.792, std dev:  12.7
Query 1115 artists: Range: 567.217 ... 605.762 ms, Average: 592.508, median: 594.644, std dev:   8.7
                    Range: 508.715 ... 543.284 us/row, Average: 531.397, median: 533.313, std dev:   7.8
Index by artist:    Range:  20.730 ...  28.636 ms, Average:  23.456, median:  23.391, std dev: 0.442
                    Range:   1.701 ...   2.349 us/doc, Average:   1.924, median:   1.919, std dev: 0.0363
Query 1115 artists: Range:  26.399 ...  28.078 ms, Average:  27.316, median:  27.377, std dev: 0.297
                    Range:  23.677 ...  25.182 us/row, Average:  24.498, median:  24.554, std dev: 0.266
Query 1887 albums:  Range:  82.016 ...  85.412 ms, Average:  83.123, median:  83.083, std dev:  0.24
                    Range:  73.557 ...  76.603 us/artist, Average:  74.550, median:  74.514, std dev: 0.216
FTS indexing:       Range: 192.383 ... 195.867 ms, Average: 194.300, median: 194.558, std dev: 0.636
                    Range:  15.783 ...  16.069 us/doc, Average:  15.941, median:  15.962, std dev: 0.0522
FTS query:          Range:  10.056 ...  11.166 ms, Average:  10.867, median:  11.018, std dev: 0.227
                    Range: 372.461 ... 413.548 us/row, Average: 402.465, median: 408.082, std dev:  8.42
```

### Android (Xamarin)

(Nexus 5, Android AOSP API 25 OS; July 19, 2017)
```
Import 12189 docs:  Range: 20.209 ... 21.585 sec, median: 21.344, std dev: 0.504
                    Range: 1.658 ... 1.771 ms/doc, median: 1.751, std dev: 0.0413
Update 1223 docs:   Range: 4.521 ... 4.750 sec, median: 4.604, std dev: 0.079
                    Range: 3.697 ... 3.884 ms/update, median: 3.765, std dev: 0.0646
Query 1115 artists: Range: 245.263 ... 253.975 ms, median: 250.236, std dev: 3
                    Range: 219.967 ... 227.781 us/row, median: 250.236, std dev: 2.69
Index by artist:    Range: 135.851 ... 153.619 ms, median: 148.694, std dev: 6
                    Range: 11.145 ... 12.603 us/doc, median: 12.199, std dev: 0.492
Query 1115 artists: Range: 178.961 ... 188.075 ms, median: 180.198, std dev: 3
                    Range: 160.503 ... 168.677 us/row, median: 161.613, std dev: 2.69
Query 1887 albums:  Range: 409.295 ... 456.837 ms, median: 427.659, std dev: 17
                    Range: 367.080 ... 409.719 us/artist, median: 383.551, std dev: 15.2
FTS indexing:       Range: 1.092 ... 1.108 sec, median: 1.096, std dev: 0.006
                    Range: 40.462 ... 41.051 ms/doc, median: 40.598, std dev: 0.222
FTS query:          Range: 26.118 ... 31.231 ms, median: 26.594, std dev: 2
                    Range: 0.967 ... 1.157 ms/row, median: 0.985, std dev: 0.0741
```

### Mac OS

(MacBook Pro (15", late 2013), 2.3GHz Intel Core i7, 16GB RAM, internal Apple SSD, macOS 10.12.6; June 21, 2017)
```
Import 12189 docs:  Range:   1.210 ...   1.238 sec, Average:   1.225, median:   1.227, std dev: 0.00516
                    Range:  99.273 ... 101.571 us/doc, Average: 100.518, median: 100.700, std dev: 0.423
Update 1223 docs:   Range: 374.795 ... 413.160 ms, Average: 387.855, median: 390.990, std dev:   5.5
                    Range: 306.456 ... 337.825 us/update, Average: 317.134, median: 319.697, std dev:  4.49
Query 1115 artists: Range:  29.121 ...  30.685 ms, Average:  30.093, median:  30.132, std dev: 0.325
                    Range:  26.118 ...  27.520 us/row, Average:  26.990, median:  27.024, std dev: 0.291
Index by artist:    Range:  18.060 ...  22.018 ms, Average:  19.092, median:  18.982, std dev: 0.384
                    Range:   1.482 ...   1.806 us/doc, Average:   1.566, median:   1.557, std dev: 0.0315
Query 1115 artists: Range:  25.413 ...  27.139 ms, Average:  26.105, median:  26.138, std dev:  0.36
                    Range:  22.792 ...  24.340 us/row, Average:  23.413, median:  23.442, std dev: 0.322
Query 1887 albums:  Range:  73.152 ...  78.027 ms, Average:  75.374, median:  75.238, std dev:  1.53
                    Range:  65.607 ...  69.980 us/artist, Average:  67.600, median:  67.478, std dev:  1.37
FTS indexing:       Range: 118.692 ... 124.893 ms, Average: 121.656, median: 121.341, std dev:  2.03
                    Range:   9.738 ...  10.246 us/doc, Average:   9.981, median:   9.955, std dev: 0.167
FTS query:          Range:   8.767 ...  10.768 ms, Average:   8.988, median:   8.955, std dev: 0.176
                    Range: 324.694 ... 398.812 us/row, Average: 332.871, median: 331.681, std dev:  6.52
```

### Windows (.NET)

(Windows 10 Home 64-bit Desktop  
Core i5 @ 3.5 Ghz  
16 GB DDR3 @ 666 Mhz (9-9-9-24)  
ASRock Z97 Extreme4 Motherboard  
Crucial MX100 256 GB SATAIII SSD drive  
July 19, 2017)
```
Import 12189 docs:  Range: 1.113 ... 1.247 sec, median: 1.142, std dev: 0.157
                    Range: 91.307 ... 102.345 us/doc, median: 93.681, std dev: 12.9
Update 1223 docs:   Range: 358.133 ... 396.489 ms, median: 361.169, std dev: 51
                    Range: 292.832 ... 324.194 us/update, median: 295.314, std dev: 41.7
Query 1115 artists: Range: 32.607 ... 36.397 ms, median: 33.983, std dev: 5
                    Range: 29.244 ... 32.643 us/row, median: 30.478, std dev: 4.48
Index by artist:    Range: 20.035 ... 26.170 ms, median: 22.895, std dev: 3
                    Range: 1.644 ... 2.147 us/doc, median: 1.878, std dev: 0.246
Query 1115 artists: Range: 20.256 ... 21.372 ms, median: 20.773, std dev: 3
                    Range: 18.167 ... 19.167 us/row, median: 18.630, std dev: 2.69
Query 1887 albums:  Range: 63.121 ... 70.197 ms, median: 67.074, std dev: 9
                    Range: 56.610 ... 62.957 us/artist, median: 60.156, std dev: 8.07
FTS indexing:       Range: 121.583 ... 133.806 ms, median: 123.274, std dev: 17
                    Range: 4.503 ... 4.956 ms/doc, median: 4.566, std dev: 0.63
FTS query:          Range: 5.377 ... 8.109 ms, median: 5.960, std dev: 1
                    Range: 199.156 ... 300.319 us/row, median: 220.748, std dev: 37
```

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