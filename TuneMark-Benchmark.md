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

(iPhone 6, iOS 11.3; April 6, 2018)
```
Import 12189 docs:  Range:   1.186 ...   1.690 sec, Average:   1.214, median:   1.203, std dev: 0.0323
                    Range:  97.300 ... 138.664 us/doc, Average:  99.631, median:  98.666, std dev:  2.65
                     Rate: 10135 docs/sec
Update 1223 docs:   Range:   1.041 ...   1.466 sec, Average:   1.062, median:   1.064, std dev: 0.00896
                     Rate: 1149 docs/sec
                    Range: 851.415 ... 1198.784 us/update, Average: 868.290, median: 870.325, std dev:  7.33
Query 1111 artists: Range:  82.111 ... 127.552 ms, Average:  83.677, median:  83.852, std dev:  1.18
                    Range:  73.907 ... 114.808 us/row, Average:  75.317, median:  75.474, std dev:  1.06
Query 1886 albums:  Range:  11.562 ...  13.504 sec, Average:  11.803, median:  11.773, std dev: 0.131
                    Range:  10.407 ...  12.155 ms/artist, Average:  10.623, median:  10.597, std dev: 0.118
Index by artist:    Range:  44.734 ...  52.680 ms, Average:  45.328, median:  45.342, std dev: 0.448
                    Range:   3.670 ...   4.322 us/doc, Average:   3.719, median:   3.720, std dev: 0.0368
Re-query artists:   Range:  53.671 ...  55.846 ms, Average:  54.413, median:  54.516, std dev: 0.282
                    Range:  48.309 ...  50.267 us/row, Average:  48.977, median:  49.069, std dev: 0.253
Re-query albums:    Range: 206.455 ... 235.331 ms, Average: 211.156, median: 211.472, std dev:  1.61
                    Range: 185.828 ... 211.819 us/artist, Average: 190.059, median: 190.344, std dev:  1.45
FTS indexing:       Range: 400.503 ... 433.702 ms, Average: 404.489, median: 404.713, std dev:  2.45
                    Range:  32.858 ...  35.581 us/doc, Average:  33.185, median:  33.203, std dev: 0.201
FTS query:          Range:   2.464 ...   3.526 ms, Average:   2.616, median:   2.603, std dev: 0.141
                    Range:  82.122 ... 117.543 us/row, Average:  87.209, median:  86.757, std dev:   4.7
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
(Nexus 6P, Android 8.1.0 API 27; April 09, 2018)
```
Import 12189 docs:  Range:   5.346 ...   6.588 sec, Average:   5.534, median:   5.388, std dev: 0.360
                    Range: 438.628 ... 540.460 µs/doc, Average: 453.996, median: 442.000, std dev:  29.6
                    Rate: 2262 docs/sec
Update 1223 docs:   Range:   5.214 ...   6.178 sec, Average:   5.381, median:   5.305, std dev: 0.271
                    Rate: 231 docs/sec
                    Range:   4.263 ...   5.052 ms/update, Average:   4.400, median:   4.338, std dev: 0.221
Query 1111 artists: Range: 184.202 ... 204.952 ms, Average: 194.568, median: 196.317, std dev:  5.96
                    Range: 165.799 ... 184.475 µs/row, Average: 175.129, median: 176.703, std dev:  5.36
Query 1886 albums:  Range:  36.478 ...  37.615 sec, Average:  37.058, median:  37.119, std dev: 0.328
                    Range:  32.833 ...  33.857 ms/artist, Average:  33.355, median:  33.410, std dev: 0.295
Index by artist:    Range: 105.114 ... 265.312 ms, Average: 171.114, median: 113.130, std dev:  74.7
                    Range:   8.624 ...  21.767 µs/doc, Average:  14.038, median:   9.281, std dev:  6.13
Re-query artists:   Range: 129.608 ... 134.652 ms, Average: 131.722, median: 131.358, std dev:  1.95
                    Range: 116.659 ... 121.199 µs/row, Average: 118.562, median: 118.234, std dev:  1.76
Re-query albums:    Range: 435.908 ... 636.117 ms, Average: 516.345, median: 509.667, std dev:  47.2
                    Range: 392.356 ... 572.563 µs/artist, Average: 464.757, median: 458.746, std dev:  42.5
FTS indexing:       Range: 693.372 ... 721.314 ms, Average: 707.403, median: 708.698, std dev:  7.32
                    Range:  56.885 ...  59.177 µs/doc, Average:  58.036, median:  58.142, std dev: 0.601
FTS query:          Range:   4.504 ...   5.971 ms, Average:   5.429, median:   5.734, std dev: 0.498
                    Range: 150.130 ... 199.049 µs/row, Average: 180.981, median: 191.127, std dev:  16.6
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

(MacBook Pro (15", late 2013), 2.3GHz Intel Core i7, 16GB RAM, internal Apple SSD, macOS 10.13.3; April 6, 2018)
```
Import 12189 docs:  Range: 426.578 ... 485.405 ms, Average: 454.841, median: 462.388, std dev:  9.69
                    Range:  34.997 ...  39.823 us/doc, Average:  37.316, median:  37.935, std dev: 0.795
                     Rate: 26361 docs/sec
Update 1223 docs:   Range: 320.336 ... 363.441 ms, Average: 334.622, median: 332.976, std dev:  8.32
                     Rate: 3673 docs/sec
                    Range: 261.927 ... 297.171 us/update, Average: 273.608, median: 272.262, std dev:   6.8
Query 1111 artists: Range:  27.942 ...  32.751 ms, Average:  30.006, median:  30.812, std dev:  1.49
                    Range:  25.151 ...  29.479 us/row, Average:  27.008, median:  27.734, std dev:  1.34
Query 1886 albums:  Range:   3.354 ...   3.756 sec, Average:   3.475, median:   3.497, std dev: 0.0421
                    Range:   3.019 ...   3.381 ms/artist, Average:   3.128, median:   3.147, std dev: 0.0379
Index by artist:    Range:  14.023 ...  16.721 ms, Average:  15.559, median:  15.648, std dev: 0.604
                    Range:   1.150 ...   1.372 us/doc, Average:   1.276, median:   1.284, std dev: 0.0495
Re-query artists:   Range:  16.608 ...  19.515 ms, Average:  17.893, median:  18.060, std dev:  0.58
                    Range:  14.949 ...  17.566 us/row, Average:  16.106, median:  16.256, std dev: 0.522
Re-query albums:    Range:  65.792 ...  80.442 ms, Average:  71.221, median:  71.276, std dev:  3.84
                    Range:  59.219 ...  72.405 us/artist, Average:  64.106, median:  64.154, std dev:  3.46
FTS indexing:       Range:  90.588 ... 110.644 ms, Average:  96.574, median:  95.494, std dev:   5.4
                    Range:   7.432 ...   9.077 us/doc, Average:   7.923, median:   7.834, std dev: 0.443
FTS query:          Range: 731.911 ... 1809.565 us, Average: 803.707, median: 766.579, std dev:  79.6
                    Range:  24.397 ...  60.319 us/row, Average:  26.790, median:  25.553, std dev:  2.65
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