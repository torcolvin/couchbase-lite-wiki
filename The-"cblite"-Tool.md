`cblite` is a command-line tool for inspecting and querying LiteCore and Couchbase Lite databases. It has three sub-commands:

| Command | Purpose |
|---------|---------|
| `cblite file` | Display information about the database |
| `cblite ls` | List the documents in the database |
| `cblite query` | Run queries, using the [[JSON query syntax|JSON Query Schema]] |

## Example

```
$  cblite file travel-sample.cblite2
Database:   travel-sample.cblite2/
Total size: 34MB
Documents:  31591, last sequence 31591

$  cblite ls -l --limit 10 travel-sample.cblite2
Document ID     Rev ID     Flags   Seq     Size
airline_10      1-d70614ae ---       1     0.1K
airline_10123   1-091f80f6 ---       2     0.1K
airline_10226   1-928c43f4 ---       3     0.1K
airline_10642   1-5cb6252c ---       4     0.1K
airline_10748   1-630b0443 ---       5     0.1K
airline_10765   1-e7999661 ---       6     0.1K
airline_109     1-bd546abb ---       7     0.1K
airline_112     1-ca955c69 ---       8     0.1K
airline_1191    1-28dbba6e ---       9     0.1K
airline_1203    1-045b6947 ---      10     0.1K
(Stopping after 10 docs)

$  cblite query --limit 10 travel-sample.cblite2 '["=", [".type"], "airline"]'
["_id": "airline_10"]
["_id": "airline_10123"]
["_id": "airline_10226"]
["_id": "airline_10642"]
["_id": "airline_10748"]
["_id": "airline_10765"]
["_id": "airline_109"]
["_id": "airline_112"]
["_id": "airline_1191"]
["_id": "airline_1203"]
(Limit was 10 rows)

$  cblite query --limit 10 travel-sample.cblite2 '{WHAT: [[".name"]], WHERE:  ["=", [".type"], "airline"], ORDER_BY: [[".name"]]}'
["40-Mile Air"]
["AD Aviation"]
["ATA Airlines"]
["Access Air"]
["Aigle Azur"]
["Air Austral"]
["Air Caledonie International"]
["Air CaraÃ¯bes"]
["Air Cargo Carriers"]
["Air Cudlua"]
(Limit was 10 rows)
```

## Parameters

(You can run `cblite --help` to get a quick summary.)

### file

`cblite file` _databasepath_

### ls

`cblite ls` _[flags]_ _databasepath_

| Flag    | Effect  |
|---------|---------|
| `-l` | Long format (one doc per line, with metadata) |
| `--offset` _n_ | Skip first _n_ docs |
| `--limit` _n_ | Stop after _n_ docs |
| `--desc` | Descending order |
| `--seq` | Order by sequence, not docID |
| `--del` | Include deleted documents |
| `--conf` | Include _only_ conflicted documents |

### query

`cblite query` _[flags]_ _databasepath_ _query_

| Flag    | Effect  |
|---------|---------|
| `--offset` _n_ | Skip first _n_ rows |
| `--limit` _n_ | Stop after _n_ rows |

The _query_ must follow the [[JSON query schema|JSON Query Schema]]. [JSON5](http://json5.org) syntax is allowed. It can be a dictionary {`{ ... }`) containing an entire query specification, or an array (`[ ... ]`) with just a `WHERE` clause. There are examples of each up above.