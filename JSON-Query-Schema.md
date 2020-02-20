## Table Of Contents

* [Introduction][1]
* [Example][2]
* [Leaf Types][3]
* [Operations][4]
* [Collation][5]
* [`MATCH` and Full-Text Search][6]
* [Top-Level Query, and `SELECT`][7]
* [Functions][8]
* [Indexes](#9-Indexes)

>**STATUS:** This is a living document that reflects the latest version of the query syntax, as implemented on the master branch of LiteCore. It may describe features not yet available in a release of Couchbase Lite. Pay attention to "STATUS:" blocks like this one, which point out new features not yet in a release.

## 1. Introduction

Couchbase Lite's query builder API generates an intermediate representation of the query, which is then given to LiteCore to translate to SQL and execute. That representation is expressed as a JSON schema; this document describes it.

>**STATUS:** At this time (Nov 2019) it is not possible to use this JSON syntax directly in Couchbase Lite, except for [Couchbase Lite For C][CBL_C].

A query is described in JSON as a sort of parse tree. Each **node** of the tree describes an **operation** and a list of **operands** (children). The operations can be arithmetic, comparison, logical, etc. The number of operands depends on the operation; for example, `NOT` has exactly one, `-` has one or two (negation or subtraction), `AND` has two or more.

A node is represented in JSON as an array, where the first element is a string naming the operation, and the other elements represent the operands (often nested arrays); for example `["=", ["+", 2, 2], 5]`. (If you know LISP or any functional languages, this should look pretty familiar!)

A few operations, like `SELECT` and `COLLATE`, take a set of _named_ operands. These are represented as a single operand that's a JSON object/dictionary whose items are the real operands.

Most of the operation names are SQL / N1QL keywords or math symbols, but there are also operations to represent document property paths (`"."`), query parameters (`"$"`), etc.

**NOTE:** This schema is case-insensitive, like SQL and N1QL. All operation names, function names, and `SELECT` keys can be upper- or lower-case or any mixture.

## 2. Example

`SELECT name.first, name.last FROM students WHERE grade = 12 AND gpa >= $GPA`

As a JSON tree this looks like:

```
["SELECT", {
    "WHAT": [
        [".", "name", "first"],
        [".", "name", "last"] ],
    "WHERE":
        ["AND",
            ["=",
                [".", "grade"],
                12],
            [">=",
                [".", "gpa"],
                ["$", "GPA"] ] } ]
```

## 3. Leaf Types (Literals, Properties, Parameters, Variables)

### Literals

As an operand, a JSON string, number, boolean or `null` represents itself.

Examples: `true`, `false`, `null`, `17`, `"foo"`

The special value `MISSING` (equivalent to SQL's `NULL`) is represented as `["MISSING"]`; i.e. it's a zero-argument operation that returns a constant.

An array literal is created using the `"[]"` operation. All of the operands are evaluated and concatenated to make the array.

Example: `["[]", 10, true, "foo"]`

A dictionary literal is created simply by using a JSON dictionary/object. All of the values are evaluated.

Example: `{"name": [".name"], "age": [".age"]}`

**NOTE:** Array and dictionary literals ignore any values that evaluate to `MISSING` (similar to JavaScript's `undefined`.) So in the dictionary example above, if the document had no `age` property, the resulting dictionary would have only a `name` key.

### Properties

Properties are references to document properties. The property operation name is `"."`; its operands are a path from the document root to the property being named.

Example:  `[".", "name", "first"]`

As shorthand, a property expression can be collapsed into a one-element array, like `[".name.first"]` ... as long as none of the path components contain a "`.`", of course.

A property expression with zero operands, `["."]`, represents the root of the document. (This is commonly used in a `WHAT` list, where it is the equivalent of the SQL `*` specifier.)

In a query with a `FROM` clause, where multiple documents are being queried, a property expression's path MUST be prefixed with the alias of the document as its first operand. For example, if the alias were `db`, then `[".", "name"]` would become `[".", "db", "name"]`; `["."]` would become `[".", "db"]`; and `[".", "_id"]` would become `[".", "db", "_id"]`. Of course these can be abbreviated as `[".db.name"]`, etc.

If the query's `WHAT` clause uses the `AS` operator to declare a [result alias][RESULTALIASES], that alias can be used as a top-level property name (potentially hiding a document property with the same name!)

#### Magic Metadata Properties

There are some special top-level property names for accessing document metadata:

| Name | Type | Value |
|------|------|-------|
| `_id` | string | The document ID |
| `_sequence` | integer | The sequence number |
| `_deleted` | boolean | True if the document is deleted |
| `_expiration` | integer or missing | Expiration time (ms since Unix epoch) |

### Parameters

Parameters are placeholders whose values are substituted when the query is run. The parameter name is the single (required) operand of the `"$"` expression.

Example: `["$", "MIN_AGE"]`

As shorthand, a parameter expressions can be collapsed into a one-element array, like `["$MIN_AGE"]`.

### Variables

Variables are placeholders used in a `ANY` and `EVERY` expression to represent the collection item being iterated over. The (required) first operand of the `"?"` expression is the variable's name, and the (optional) extra operands are a property path relative to the variable's value.

Example: `["?", "address", "zip"]`

As shorthand, a variable expression can be collapsed into a one-element array, like `["?address"]` or `["?address.zip"]`.

## 4. Operations

The operations are named after their N1QL/SQL equivalents.

|Category| Name | Operand Count |
|--------|------|---------------|
| Constants | `MISSING` | 0 |
|Arithmetic| `+`  | 2+ |
| | `-`  | 1 or 2 |
| | `*`  | 2+ |
| | `/`  | 2 |
| | `%`  | 2 |
|String| `\|\|`  | 2+ |
|Relational| `=` | 2 |
| | `!=` | 2 |
| | `<` | 2 |
| | `<=` | 2 |
| | `>` | 2 |
| | `>=` | 2 |
| | `BETWEEN` | 3: (value, min, max) |
| | `IS` | 2 |
| | `IS NOT` | 2 |
| | `LIKE` | 2 |
| | `MATCH` | 2 |
| | `IN` | 2: (value, array) |
| | `NOT IN` | 2: (value, array) |
| | `EXISTS` | 1 |
| | `COLLATE` | 2: (options, expr) [see **[Collation][10]** below] |
|Logical| `NOT` | 1 |
| | `AND` | 2+ |
| | `OR` | 2+ |
|Functions| _name_`()` | Depends on [function][11] |
|Conditional| `CASE` | 3+: (expr, when1, then1 ...) [see [below][CASE]] |
| | `ANY` | 3: (variable name, array, satisfies) |
| | `EVERY` | 3: (variable name, array, satisfies) |
| | `ANY AND EVERY` | 3: (variable name, array, satisfies) |
|Properties| `.` | 0+: (path components) [[see above][12]] |
| | `_.` | 2: (expr, property-path) Evaluates a property (or path) of a dictionary value |
|Parameters| `$` | 1 (name or position) [[see above][13]] |
|Variables| `?` | 1+ (name, optional path components) [[see above][14]] |
|Blobs| `BLOB` | 1: (property path) |
|Queries| `SELECT` | 1 [[see below][15]] |

### 4.1. CASE

The `CASE` operator needs a bit of explanation.

* The first operand is the expression to test (which directly follows the `CASE` keyword in N1QL/SQL), or `null` if there is none.
* The second operand is the first expression to compare with (directly following `WHEN`.)
* The third operand is the result to use if the first expression matches (directly following `THEN`.)
* After that can come zero or more pairs of extra 'when' and 'then' expressions.
* If there's one operand left over (i.e. the operation has an even number of operands) it's interpreted as the `ELSE` result.

## 5. Collation

The `COLLATE` operator does nothing itself, merely returns the value of its second operand, but it alters the string collation (comparison/sorting) used when evaluating that expression _and nested expressions_.

**STATUS:** (Nov 2019) The regular-expression functions don't yet obey collations. ([^296](https://github.com/couchbase/couchbase-lite-core/issues/296))

The first operand is a dictionary that specifies the collation; its keys are:

| Key | Value | Default Value |
|-----|-------|---------------|
| `"UNICODE":` | Unicode-aware? | `false` |
| `"CASE":` | Case-sensitive? | `true` |
| `"DIAC":` | Diacritic (accent) -sensitive? | `true` |
| `"LOCALE":` | ISO locale\* string or `null` | `null` |

\* A **locale** is an ISO-639 [language code][16] plus, optionally, an underscore and an ISO-3166 [country code][17]: `"en"`, `"en_US"`, `"fr_CA"`, etc.

Some details on combining keys:

* If `UNICODE` is not true, only the `CASE` value is significant: a case-sensitive collation is a purely binary string comparison; a case-insensitive one also treats ASCII uppercase and lowercase letters as equivalent.
* If `UNICODE` is true, but `LOCALE` is missing or null, the collation is Unicode-aware but not localized; for example, accented Roman letters sort right after the base letter. (This is implemented by using the "en\_US" locale.)
* Any keys not specified are inherited from the enclosing context.
* There's implicitly a top-level context with the default values for the keys, i.e. `{UNICODE: false, CASE: true, DIAC: true, LOCALE: null}`.

### About Unicode Collation

The details of [Unicode collation][18] are quite complex, though for the most part it just Does The Right Thing according to a human of that locale. But it doesn't behave like the simple `strcmp` and `strcasecmp` functions that programmers are used to!

* The collation algorithm first compares the strings ignoring case and diacritics, just looking at the base letters. If the letters are not equal, it stops and returns the relative ordering based on the mismatched letters. This is usually, but _not always_, the ordering English speakers are used to; for example, in Lithuanian "Y" comes after "I", and in traditional Spanish the sequence "CH" sorts as a single letter that comes between "C" and "D".
* Otherwise, if diacritic-sensitive, it compares the strings again, this time considering also diacritics (accents). If they differ, it returns the relative ordering. By default an accented letter sorts just after the base letter, but many locales have special rules: "Å" in Danish is treated as a separate letter that sorts just after "Z".
* Otherwise, if case-sensitive, it compares the strings again, now considering case; if they differ in case, it returns the relative ordering, with lowercase coming _before_ uppercase (contrary to the intuition of programmers used to ASCII ordering!)

The above applies to the Roman alphabet. For many non-Roman scripts, especially Chinese and Japanese, the rules get much more complicated.

Some unintuitive results: in a case-sensitive collation, "abc" comes before "ABC" (lowercase first!), but "abd" comes _after_ "ABC" because the letter mismatch takes priority over the case mismatch. Likewise, "ápple" comes after "Apple" (in most locales) because the diacritic is higher priority than uppercase.

## 6. `MATCH` and Full-Text Search

The `MATCH` operator queries a full-text-search (FTS) index. 

* Its _first_ parameter is the name of the index.
* Its _second_ parameter is a search string.

SQLite imposes limitations on how `MATCH` expressions can be used:

* They have to appear at the top level of a `SELECT`.
* Each index can only be matched once within any `SELECT`.

### FTS Search-String Syntax

The search string in a `MATCH` expression is (currently) passed directly to SQLite’s FTS4 search engine, so its [syntax][19] is specified by FTS4. A summary:

* Search terms (words) are separated by whitespace.
* If there are multiple terms, by default the text must contain all of them.
* A term ending in `*` will match any word beginning with that prefix.
* A term starting with “`^`” only matches at the start of the text. 
* A series of terms enclosed in double-quotes is a **phrase**; the terms must appear adjacent to each other in that order in the text.
* Parentheses can be used to group terms/phrases into a larger term.
* The special words `OR`, `AND`, and `NOT` (capitalized!) can be used between terms/phrases as boolean operators.
* The special word `NEAR` can be used between two terms/phrases to specify that they must appear near each other: within 10 words by default, but you can customize this by appending a “`/`” and a number, e.g. `NEAR/5`.

### Word Matching

Full-text search is _always_ case-insensitive. Its diacritic sensitivity (whether it ignores accent marks) is configured when the FTS index is created. Thus, a `MATCH` expression is not affected by a `COLLATE` clause.

Matching is affected by stemming and stop-words, if those are available in the selected language and enabled in the index.

* **Stemming** causes different forms of the same word to match, so (in English) “bigger” matches “big” and “biggest”.
* **Stop-words** are common but low-significance words, like English “the” and “are”, that are ignored completely in order to keep down the size of the index.

**STATUS:** (Nov 2019) Stemming is currently available for Danish, Dutch, English, Finnish, French, German, Hungarian, Italian, Norwegian, Portuguese, Romanian, Russian, Spanish, Swedish, Turkish. Stop-words are used in English and French.

**STATUS:** (Nov 2019) The FTS indexer considers words to be sequences of Unicode alphabetic characters separated by non-alphabetic characters. This is true of most languages, but many Asian languages like Japanese, Chinese and Thai do not normally use whitespace to separate words; FTS will not work with such text. (Finding word breaks in these languages is difficult and will require 3rd party libraries like [Mecab][20] or Apple’s [NSLinguisticTagger][21].)

## 7. Top-Level Query, and `SELECT`

The `SELECT` statement has so many parameters, all of which are optional, that it makes a lot more sense to encode them as a dictionary with the following keys:

| Key | Value | Default Value |
|-----|-------|---------------|
| `"WHAT":` | Array of [column expressions](#the-what-clause-result-columns) to return, generally properties | document ID and sequence |
| `"FROM":` | Array of [database/join identifiers](#the-from-clause-databasejoinunnest-identifiers) | Database being queried |
| `"WHERE":` | Boolean-valued expression | `true` (all documents) |
| `"HAVING":` | Expression | `true` |
| `"DISTINCT":` | Boolean | `false` |
| `"GROUP_BY":` | Array of expressions or property names | `[]` (no grouping) |
| `"ORDER_BY":` | Array of expressions or property names | `[]` (unsorted) |
| `"LIMIT":` | Number | Infinite |
| `"OFFSET":` | Number | 0 |

### The `WHAT` Clause: Result Columns

The `WHAT` array defines the columns of a query row, just like the column expressions after `SELECT` in N1QL/SQL syntax. Each item of the array may be:

* A string literal, interpreted as a property path.
* An expression (array or dictionary). Aggregate functions are allowed here.
* An array of the form `["AS", <expression>, "<string>"]`, which is interpreted exactly the same as `<expression>`, but has the side effects of defining `<string>` as an **result alias** for `<expression>`, and setting the column's **title** to `<string>`.

#### Result Aliases

A result alias can be used in the `WHERE` clause as a shortcut for its expression, as though it were a document property. (This happens even if there is a document property with the same name; the alias shadows it.)

Example: `{"WHAT":[["AS", ["+", [".x"], [".y"]], "sum"]], "WHERE": [">", [".sum"], 10]}`

**STATUS:** (Feb 2020): Result aliases were added in version 2.7.

#### Column Titles

Column titles have no effect on the query but are returned via the `c4query_getColumnTitle()` C function. A column declared without `AS` has a default title based on the property path if any, or otherwise a made-up name like `$1`. Higher-level bindings may support accessing a query row as a dictionary using the column titles as keys.

### The `FROM` Clause: Database/Join/Unnest Identifiers

The items in the `FROM` array are dictionaries with the following keys:

| Key | Value | Default Value |
|-----|-------|---------------|
| `"AS":` | Alphanumeric string: an alias to refer to this database or join by | _required_ |
| `"DB":` | String: Database name | Database being queried |
| `"JOIN":` | String: Type of join | `"INNER"` (if `ON` is given) |
| `"ON":` | Boolean-valued expression: the join constraint | no join |
| `"UNNEST":` | Array-valued expression | no unnest |

Some requirements:
* Every item must have a unique `AS` property value.
* The first item in the array serves only to alias the default database; it can't have a `JOIN`,  `ON` or `UNNEST` property.
* The subsequent items _must_ either be joins or unnests.
* In a join:
    * There must be an `ON` property.
    * Legal values for `JOIN` are `"INNER"` (the default), `"OUTER"`, `"LEFT OUTER"`, and `"CROSS"`. (Case-insensitive)
* In an unnest:
    * There must be an `UNNEST` property.
    * There cannot be a `JOIN` or `ON` property.

**STATUS:** (Nov 2019) The `DB` property is not yet implemented; only one database can be queried at a time.

Example:
```json
    "FROM": [{"as": "person"},
             {"as": "state", "on": ["=", [".state.abbreviation"],
                                         [".person.contact.address.state"]]},
             {"as": "interest", "unnest": [".person.interests"]}],
```

**Adding a `FROM` clause affects the interpretation of properties.** Since there are usually multiple databases or join sources, property names (paths) in the entire query need to be disambiguated by prefixing the appropriate alias. So in the above example, a document's `abbreviation` property has to be named as `.state.abbreviation`, not just `.abbreviation`.

## 8. N1QL Functions

For detailed information about parameters and results, please consult the [N1QL documentation][23]. 

**NOTE:** There are some differences from SQL, or at least from SQLite; for example, SQLite has non-aggregate versions of `min` and `max`, but in N1QL (and LiteCore) these are called `least` and `greatest`.

|Category| Name | Operand Count | Notes |
|--------|------|---------------|-------|
| **Aggregate** | `array_agg()` | 1 | Collects values into an array |
| | `avg()` | 1 | |
| | `count()` | 1 | |
| | `max()` | 1 | |
| | `min()` | 1 | |
| | `sum()` | 1 | |
| **Arrays** | `array_avg()` | 1 | Average value of numbers in an array |
| | `array_contains()` | 2 | (_array_, _item_) |
| | `array_count()` | 1 | Number of non-null items in array |
| | `array_ifnull()` | 1 | The first non-null item in an array |
| | `array_length()` | 1 | Full length of array |
| | `array_max()` | 1 | Maximum number in an array |
| | `array_min()` | 1 | Minimum number in an array |
| | `array_sum()` | 1 | |
| **Comparisons** | `greatest()` | 2+ | Maximum numeric argument (like SQL `max`) |
| | `least()` | 2+ | Minimum numeric argument (like SQL `min`)  |
| **Conditionals** | `ifmissing()` | 2+ | Returns 1st non-`missing` arg |
| | `ifmissingornull()` | 1+ | Returns 1st arg not `missing` or `null` |
| | `ifnull()` | 1+ | Returns 1st non-`null` arg |
| | `missingif()` | 2 | returns `missing` if arg 1 == arg 2, else returns arg 1 |
| | `nullif()` | 2 | returns `null` if arg 1 == arg 2, else returns arg 1 |
| **Dates** | `millis_to_str()` | 1 | Converts Unix timestamp in milliseconds to ISO-8601 date string in _local_ time zone |
| | `millis_to_utc()` | 1 | Converts Unix timestamp in milliseconds to ISO-8601 date string in UTC |
| | `str_to_millis()` | 1 | Parses ISO-8601 date string to Unix timestamp in milliseconds |
| | `str_to_utc()` | 1 | Normalizes ISO-8601 date string to UTC timezone |
| **Math** | `abs()` | 1 | |
| | `acos()` | 1 | Note: All trig functions use radians |
| | `asin()` | 1 | |
| | `atan()` | 1 | |
| | `atan2()` | 2 | |
| | `ceil()` | 1 | |
| | `cos()` | 1 | |
| | `degrees()` | 1 | Converts radians to degrees |
| | `e()` | 0 | |
| | `exp()` | 1 | |
| | `ln()` | 1 | |
| | `log()` | 1 | |
| | `floor()` | 1 | |
| | `pi()` | 0 | |
| | `power()` | 2 | |
| | `radians()` | 1 | Converts degrees to radians |
| | `round()` | 1–2 | Optional 2nd argument gives number of decimal places to round to (default 0) |
| | `sign()` | 1 | Returns -1, 0 or 1, reflecting sign of argument |
| | `sin()` | 1 | |
| | `sqrt()` | 1 | |
| | `tan()` | 1 | |
| | `trunc()` | 1–2 | Optional 2nd argument gives number of decimal places to truncate to (default 0) |
| **Patterns** | `regexp_contains()` | 2 | Args are (_string_, _pattern_) |
| | `regexp_like()` | 2 | Synonym for `regexp_contains` |
| | `regexp_position()` | 2 | Returns byte offset of 1st match, else -1 |
| | `regexp_replace()` | 3-4 | Args are (_string_, _pattern_, _replacement_) and optional _limit_ |
| | `rank()` | 1 | Returns ranking of FTS matches |
| **Strings** | `contains()` | 2 | |
| | `concat()` | 2+ | |
| | `length()` | 1 | |
| | `lower()` | 1 | |
| | `ltrim()` | 1 | Removes leading whitespace |
| | `rtrim()` | 1 | Removes trailing whitespace |
| | `trim()` | 1–2 | Removes leading & trailing whitespace |
| | `upper()` | 1 | |
| **Types** | `isarray()` | 1 | |
| | `isatom()` | 1 | "Atom" means boolean, number, or string |
| | `isboolean()` | 1 | |
| | `isnumber()` | 1 | |
| | `isobject()` | 1 | |
| | `isstring()` | 1 | |
| | `type()` | 1 | Returns one of `'missing'`, `'null'`, `'boolean'`, `'number'`, `'string'`, `'binary'`, `'array'`, `'object'` |
| | `toatom()` | 1 | See N1QL docs |
| | `toboolean()` | 1 | See N1QL docs |
| | `tonumber()` | 1 | See N1QL docs |
| | `tostring()` | 1 | See N1QL docs |
| **Predictive** | `prediction()` [q.v.] | 2-3 | |
| | `euclidean_distance()` | 2-3 | |
| | `cosine_distance()` | 2 | |

**STATUS:** (Nov 2019) The `concat()` function is a new addition. It is equivalent to the `||` operator but can take more than two parameters.

### `prediction()`

`prediction()` is not standard N1QL. It calls a **predictive** function, usually based on a machine-learning model, which must be registered with LiteCore at runtime before the query is compiled. Its parameters are:

1. The name the predictivefunction was registered with
2. The function's named inputs; this must be a dictionary and is usually given as a dictionary literal
3. The name of the function output to use [optional]

If parameter 3 is not given, the function's output will be returned as a dictionary; otherwise only the named output value is returned.

If the model is unable to process the input because a required parameter is missing, or a parameter has the wrong type, the result of the function call is `MISSING`.

Example:

```["prediction()", "mobilenet", {"image": ["BLOB", ".picture"]}, "classLabel"]```

Assuming a trained [MobileNet image classifier][MOBILENET] has been registered as `"mobilenet"`, this will read an attached blob from the document's `picture` property, run it through MobileNet to classify it, and return the label most likely to apply to the image contents, e.g. "siamese cat" or "banana".

### `euclidean_distance()`

Returns the [_Euclidean distance_][EUCLIDEAN] between two vectors, which is used as a distance metric in predictive queries.

Both parameters must be arrays of numbers, and must be the same length. The result is a non-negative floating-point number.

An optional third parameter is a power to raise the result to. Using `2` provides the common "squared Euclidean" distance.

### `cosine_distance()`

Returns the _cosine distance_ (one minus the [_cosine similarity_][COSINE]) between two vectors, which is used as a distance metric in predictive queries.

Both parameters must be arrays of numbers, must be the same length, and must be non-empty. The result is a floating-point number in the range [-1 … +1].

**Note:** `prediction()`, `euclidean_distance()`, and `cosine_distance()` are only available in the Enterprise Edition (EE) of Couchbase Lite.


## 9. Indexes

Indexes aren't, strictly speaking, part of queries, but they use similar syntax. An index specifier is a JSON dictionary with the following keys:

| Key  | Required? | Value |
|------|-----------|-------|
| WHAT | Yes | Array of one or more expressions to index, generally document properties. |
| WHERE | No | Expression that returns a "truthy" value for the documents to be indexed. |

For backward compatibility an index specifier may also be an array, which is interpreted as though it were the value of a `WHAT` clause.

**STATUS:** (Nov 2019) The `WHERE` clause, and the dictionary form of the specifier, are experimental. In all current releases the specifier _must_ be an array.

The effective use of indexes to optimize queries is sort of a black art. Fortunately there is a lot of information in books and online, and most of that advice applies here too.

### The `WHAT` Clause

The `WHAT` clause of course determines what will be indexed. The index is sorted by these expression(s), in the same way as a query is sorted by its `ORDER BY` clause; so if there are multiple expressions, the first is the primary, the second the secondary, etc. 

The `DESC` operator can be wrapped around an expression to specify a descending sort; this can accelerate queries that use descending order or that use the `max()` function.

In a **full-text index**, the expression determines the text that will be indexed for each document, so it should evaluate to a string. (If it doesn't, that document is ignored.) Full-text indexes only support a single expression.

### The `WHERE` Clause

The optional `WHERE` clause creates a _partial index_ that includes only some of the documents in the database. A partial index is smaller, and faster to create/update, but it can only be used by a query whose own `WHERE` clause contains an equivalent condition.

Since many real-world queries look for only a particular type of document, indexes used by such queries can take advantage of a `WHERE` clause that tests the document type. For example, an index of flight arrival times might look like `{"WHAT": [[".arrival_time"]], "WHERE": ["=", [".type"], "flight"]}`.

**Full-text indexes** do not support `WHERE` clauses (yet).

[1]:	#1-introduction
[2]:	#2-example
[3]:	#3-leaf-types-literals-properties-parameters-variables
[4]:	#4-operations
[5]:	#5-collation
[6]:	#6-match-and-full-text-search
[7]:	#7-top-level-query-and-select
[8]:	#8-n1ql-functions
[9]:	http://www.sqlite.org/lang_expr.html
[10]:	#collation
[11]:	#functions
[12]:	#properties
[13]:	#parameters
[14]:	#variables
[15]:	#top-level-query-and-select
[16]:	https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes
[17]:	https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2
[18]:	http://userguide.icu-project.org/collation
[19]:	https://sqlite.org/fts3.html#full_text_index_queries
[20]:	https://github.com/jordwest/mecab-docs-en
[21]:	https://developer.apple.com/documentation/foundation/nslinguistictagger/tokenizing_natural_language_text
[22]:	#databasejoin-identifiers
[23]:	https://developer.couchbase.com/documentation/server/4.5/n1ql/n1ql-language-reference/functions.html
[CASE]: #41-case
[RESULTALIASES]: #result-aliases
[MOBILENET]: https://ai.googleblog.com/2017/06/mobilenets-open-source-models-for.html
[EUCLIDEAN]: https://en.wikipedia.org/wiki/Euclidean_distance
[COSINE]: https://en.wikipedia.org/wiki/Cosine_similarity
[CBL_C]: https://github.com/couchbaselabs/couchbase-lite-C
