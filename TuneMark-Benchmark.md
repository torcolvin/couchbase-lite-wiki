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

>**NOTE:** The current (15 Aug 2017) benchmark code has a bug, in that it combines the "Update play counts" and "Update artist names" times (see below) into one "Update ___ docs" step, and mistakenly counts only the number of documents updated by the second step. So in the reports below, the per-doc times for the "Update" step really should be based on 12189 + 1223 docs (and also account for iterating over all 12189 docs twice!)

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

### iOS (Xamarin)
(iPhone 6s+, iOS 10.3.2; July 19, 2017)
```
Import 12189 docs
Range: 8.770 ... 8.976 sec, median: 8.956, std dev: 0.077
Range: 719.538 ... 736.390 us/doc, median: 734.791, std dev: 6.32
Update 1223 docs
Range: 1.688 ... 1.723 sec, median: 1.692, std dev: 0.014
Range: 1.380 ... 1.409 ms/update, median: 1.384, std dev: 0.0114
Query 1115 artists
Range: 588.415 ... 614.022 ms, median: 594.432, std dev: 3
Range: 527.726 ... 550.692 us/row, median: 533.123, std dev: 8.07
Index by artist
Range: 22.336 ... 30.148 ms, median: 24.114, std dev: 3
Range: 1.832 ... 2.473 us/doc, median: 1.978, std dev: 0.246
Query 1115 artists
Range: 26.960 ... 28.830 ms, median: 28.623, std dev: 1
Range: 24.179 ... 25.856 us/row, median: 25.671, std dev: 0.897
Query 1887 albums
Range: 110.591 ... 120.504 ms, median: 115.054, std dev: 3
Range: 99.185 ... 108.075 us/artist, median: 103.188, std dev: 2.69
FTS indexing:
Range: 185.327 ... 193.505 ms, median: 189.100, std dev: 3
Range: 6.864 ... 7.167 ms/doc, median: 356.204, std dev: 37
FTS query:
Range: 9.495 ... 11.335 ms, median: 9.618, std dev: 1
Range: 351.670 ... 419.830 us/row, median: 356.204, std dev: 37
```
### Android
(Nexus 5, Android 6.0.1 API 23; March 05, 2018)
```
 Import 12189 docs:  Range:   8.299 ...   8.885 sec, Average:   8.502, median:   8.455, std dev: 0.194
                     Range: 680.878 ... 728.926 µs/doc, Average: 697.488, median: 693.698, std dev:  15.9
                      Rate: 1442 docs/sec
 Update 1223 docs:   Range:   6.916 ...   8.068 sec, Average:   7.348, median:   7.332, std dev: 0.286
                      Rate: 167 docs/sec
                     Range:   5.655 ...   6.597 ms/update, Average:   6.009, median:   5.995, std dev: 0.234
 Query 1111 artists: Range: 188.628 ... 226.216 ms, Average: 199.192, median: 192.351, std dev:  13.1
                     Range: 169.782 ... 203.615 µs/row, Average: 179.291, median: 173.133, std dev:  11.8
 Query 1886 albums:  Range:  32.399 ...  35.021 sec, Average:  33.212, median:  32.935, std dev: 0.782
                     Range:  29.162 ...  31.522 ms/artist, Average:  29.893, median:  29.645, std dev: 0.704
 Index by artist:    Range:  79.524 ... 159.922 ms, Average: 136.858, median: 148.905, std dev:  27.9
                     Range:   6.524 ...  13.120 µs/doc, Average:  11.228, median:  12.216, std dev:  2.29
 Re-query artists:   Range: 190.661 ... 200.828 ms, Average: 194.565, median: 193.455, std dev:  3.49
                     Range: 171.612 ... 180.763 µs/row, Average: 175.126, median: 174.127, std dev:  3.14
 Re-query albums:    Range:  33.247 ...  36.150 sec, Average:  33.762, median:  33.525, std dev: 0.827
                     Range:  29.925 ...  32.538 ms/artist, Average:  30.389, median:  30.175, std dev: 0.745
 FTS indexing:       Range:   1.021 ...   1.133 sec, Average:   1.039, median:   1.027, std dev: 0.0324
                     Range:  83.782 ...  92.937 µs/doc, Average:  85.216, median:  84.226, std dev:  2.65
 FTS query:          Range:   4.196 ...   5.696 ms, Average:   4.571, median:   4.473, std dev: 0.408
                     Range: 139.865 ... 189.878 µs/row, Average: 152.377, median: 149.115, std dev:  13.6
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

### Windows (.NET Core 2.0.5)

(Windows 10 Fall Creator's Update Home 64-bit Desktop  
Core i5 @ 3.5 Ghz  
16 GB DDR3 @ 666 Mhz (9-9-9-24)  
ASRock Z97 Extreme4 Motherboard  
Crucial MX100 256 GB SATAIII SSD drive  
April 3, 2018)
```
Import 12189 docs
Range: 552.439 ... 635.115 ms, median: 566.635, std dev: 82
Range: 45.323 ... 52.106 us/doc, median: 46.487, std dev: 6.73
Update 1223 docs
Range: 182.228 ... 272.655 ms, median: 197.301, std dev: 40
Range: 149.001 ... 222.939 us/update, median: 161.325, std dev: 32.7
Query 1111 artists
Range: 27.729 ... 53.074 ms, median: 34.407, std dev: 11
Range: 24.958 ... 47.771 us/row, median: 30.969, std dev: 9.9
Query 1886 albums
Range: 4.534 ... 4.694 sec, median: 4.637, std dev: 0.617
Range: 4.081 ... 4.225 ms/artist, median: 4.174, std dev: 0.555
Index by artist
Range: 18.546 ... 24.894 ms, median: 19.714, std dev: 3
Range: 1.522 ... 2.042 us/doc, median: 1.617, std dev: 0.246
Re-query artists
Range: 27.021 ... 28.552 ms, median: 28.054, std dev: 4
Range: 24.321 ... 25.699 us/row, median: 25.251, std dev: 3.6
Re-query albums
Range: 71.287 ... 75.690 ms, median: 73.252, std dev: 10
Range: 64.164 ... 68.128 us/artist, median: 65.933, std dev: 9
FTS Indexing
Range: 88.430 ... 91.601 ms, median: 89.707, std dev: 12
Range: 7.255 ... 7.515 us/doc, median: 7.360, std dev: 0.984
FTS Query
Range: 633.300 ... 939.400 us, median: 731.300, std dev: 0
Range: 21.110 ... 31.313 us/row, median: 24.377, std dev: 0
```

### Windows (.NET UWP 6.0.6)

(Windows 10 Fall Creator's Update Home 64-bit Desktop  
Core i5 @ 3.5 Ghz  
16 GB DDR3 @ 666 Mhz (9-9-9-24)  
ASRock Z97 Extreme4 Motherboard  
Crucial MX100 256 GB SATAIII SSD drive  
April 3, 2018)

```
Import 12189 docs
Range: 553.410 ... 634.646 ms, median: 574.572, std dev: 82
Range: 45.402 ... 52.067 us/doc, median: 47.139, std dev: 6.73
Update 1223 docs
Range: 185.371 ... 235.832 ms, median: 204.367, std dev: 30
Range: 151.571 ... 192.831 us/update, median: 167.103, std dev: 24.5
Query 1111 artists
Range: 30.495 ... 56.525 ms, median: 32.977, std dev: 10
Range: 27.448 ... 50.878 us/row, median: 29.682, std dev: 9
Query 1886 albums
Range: 4.675 ... 4.995 sec, median: 4.798, std dev: 0.647
Range: 4.208 ... 4.496 ms/artist, median: 4.319, std dev: 0.582
Index by artist
Range: 20.211 ... 25.085 ms, median: 22.017, std dev: 3
Range: 1.658 ... 2.058 us/doc, median: 1.806, std dev: 0.246
Re-query artists
Range: 30.455 ... 31.643 ms, median: 30.991, std dev: 4
Range: 27.412 ... 28.482 us/row, median: 27.895, std dev: 3.6
Re-query albums
Range: 82.342 ... 91.624 ms, median: 85.923, std dev: 12
Range: 74.115 ... 82.470 us/artist, median: 77.338, std dev: 10.8
FTS Indexing
Range: 116.299 ... 130.634 ms, median: 117.338, std dev: 17
Range: 9.541 ... 10.717 us/doc, median: 9.627, std dev: 1.39
FTS Query
Range: 743.000 ... 937.000 us, median: 794.500, std dev: 0
Range: 24.767 ... 31.233 us/row, median: 26.483, std dev: 0
```

### iOS — CBL 1.4
(iPhone 6s+, iOS 10.3.3; July 20, 2017)

```
Import 12189 docs:  Range:   2.909 ...   3.092 sec, Average:   2.950, median:   2.943, std dev: 0.0468
                    Range: 238.665 ... 253.654 us/doc, Average: 242.017, median: 241.426, std dev:  3.84
Update 1223 docs:   Range:   1.743 ...   2.539 sec, Average:   1.807, median:   1.798, std dev: 0.0427
                    Range:   1.425 ...   2.076 ms/update, Average:   1.477, median:   1.470, std dev: 0.0349
Query 1114 artists: Range:  82.929 ...  91.692 ms, Average:  88.331, median:  88.713, std dev:  1.62
                    Range:  74.442 ...  82.309 us/row, Average:  79.291, median:  79.634, std dev:  1.45
Index by artist:    Range: 520.720 ... 579.324 ms, Average: 545.557, median: 544.835, std dev:  12.5
                    Range:  42.720 ...  47.528 us/doc, Average:  44.758, median:  44.699, std dev:  1.02
Query 1114 artists: N/A
                    
Query 1886 albums:  Range: 262.376 ... 280.685 ms, Average: 271.212, median: 270.650, std dev:  4.25
                    Range: 235.526 ... 251.962 us/artist, Average: 243.458, median: 242.953, std dev:  3.81
FTS indexing:       Range: 683.968 ... 757.246 ms, Average: 725.306, median: 719.835, std dev:  14.3
                    Range:  56.114 ...  62.125 us/doc, Average:  59.505, median:  59.056, std dev:  1.17
FTS query:          Range:   2.658 ...   4.485 ms, Average:   2.811, median:   2.779, std dev: 0.138
                    Range:  88.607 ... 149.508 us/row, Average:  93.699, median:  92.646, std dev:  4.59
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