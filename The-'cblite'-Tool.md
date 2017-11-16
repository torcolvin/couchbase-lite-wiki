`cblite` is a command-line tool for inspecting and querying LiteCore and Couchbase Lite databases. It has the following sub-commands:

| Command | Purpose |
|---------|---------|
| `cblite file` | Display information about the database |
| `cblite ls` | List the documents in the database |
| `cblite cat` | Display the body of one or more documents |
| `cblite revs` | List the revisions of a document |
| `cblite query` | Run queries, using the [[JSON Query Schema]] |
| `cblite sql` | Run a SQLite query directly |
| `cblite help` | Display help text |

It has an interactive mode that you start by running `cblite /path/to/database`, i.e. with no subcommand. It will then prompt you for a command, which is a command line without the initial `cblite` or the database-path parameter. Enter `quit` or press Ctrl-D to exit.

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

$  cblite travel-sample.cblite2
(cblite) query --limit 10 '["=", [".type"], "airline"]'
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
(cblite) query --limit 10 '{WHAT: [[".name"]], WHERE:  ["=", [".type"], "airline"], ORDER_BY: [[".name"]]}'
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
(cblite) ^D
$
```

## Parameters

(You can run `cblite --help` to get a quick summary.)

### file

`cblite file` _databasepath_

### ls

`cblite ls` _[flags]_ _databasepath_ _[PATTERN]_

| Flag    | Effect  |
|---------|---------|
| `-l` | Long format (one doc per line, with metadata) |
| `--offset` _n_ | Skip first _n_ docs |
| `--limit` _n_ | Stop after _n_ docs |
| `--desc` | Descending order |
| `--seq` | Order by sequence, not docID |
| `--del` | Include deleted documents |
| `--conf` | Include _only_ conflicted documents |
| `--body` | Display document bodies |
| `--pretty` | Pretty-print document bodies (implies `--body`) |
| `--json5` | JSON5 syntax, i.e. unquoted dict keys (implies `--body`)|

(PATTERN is an optional pattern for matching docIDs, with shell-style wildcards `*`, `?`)

### cat

`cblite cat` _[flags]_ _databasepath_ _DOCID_ [_DOCID_ ...]

| Flag    | Effect  |
|---------|---------|
| `--key KEY` | Display only a single key/value (may be used multiple times) |
| `--rev` | Show the revision ID(s) |
| `--raw` | Raw JSON (not pretty-printed) |
| `--json5` | JSON5 syntax (no quotes around dict keys) |

(DOCID may contain shell-style wildcards `*`, `?`)

### revs

`cblite revs` _databasepath_ _DOCID_

### query

`cblite query` _[flags]_ _databasepath_ _query_

| Flag    | Effect  |
|---------|---------|
| `--offset` _n_ | Skip first _n_ rows |
| `--limit` _n_ | Stop after _n_ rows |

The _query_ must follow the [[JSON query schema|JSON Query Schema]]. [JSON5](http://json5.org) syntax is allowed. It can be a dictionary {`{ ... }`) containing an entire query specification, or an array (`[ ... ]`) with just a `WHERE` clause. There are examples of each up above.

## Where To Get It

The tool is included in Couchbase Lite 2 builds as of DB18. As of now (Nov 8 2017) it's only included in the iOS/Mac archive, but it will later be included in other platforms as well.

You can build the tool for Mac OS via the LiteCore Xcode project. Choose (or create) the `cblite` scheme and build it; then dig through the build output to find the `cblite` binary. You can move the tool anywhere; it has no external dependencies.