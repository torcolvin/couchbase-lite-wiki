## Table Of Contents

* [Introduction](#introduction)
* [Example](#example)
* [Leaf Types](#leaf-types)
* [Operations](#operations)
* [Collation](#collation)
* [Top-Level Query, and `SELECT`](#top-level-query-and-select)
* [Functions](#functions)
* [Implementation Status](#implementation-status)

## 1. Introduction

Queries are expressed to LiteCore as JSON, so they can be easily transformed and converted to internal representations like [SQL](http://www.sqlite.org/lang_expr.html). This document describes the schema.

A query is described as a sort of parse tree. Each node of the tree describes an **operation** and a list of **operands** (children). The operations can be arithmetic, comparison, logical, etc. The number of operands depends on the operation; for example, `NOT` has exactly one, `-` has one or two (negation or subtraction), `AND` has two or more.

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

Examples: `true`, `null`, `17`, `"foo"`

An array literal is created using the `"[]"` operation. All of the operands are evaluated and concatenated to make the array.

Example: `["[]", 10, true, "foo"]`

### Properties

Properties are references to document properties. The property operation name is `"."`; its operands are a path from the document root to the property being named.

Example:  `[".", "name", "first"]`

As shorthand, a property expression can be collapsed into a one-element array, like `[".name.first"]`.

A property expression with zero operands, `["."]`, represents the root of the document. (This is commonly used in a `WHAT` list, where it is the equivalent of the SQL `*` specifier.)

The special top-level property names `_id` and `_sequence` refer to the document's ID and current sequence number. _[TBD: This may change to a special `meta` object.]_

In a query with a `FROM` clause, where multiple documents are being queried, a property expression's path MUST be prefixed with the alias of the document as its first operand. For example, if the alias were `db`, then `[".", "name"]` would become `[".", "db", "name"]`; `["."]` would become `[".", "db"]`; and `[".", "_id"]` would become `[".", "db", "_id"]`. Of course these can be abbreviated as `[".db.name"]`, etc.

### Parameters

Parameters are placeholders whose values are substituted when the query is run. The parameter name is the single (required) operand of the `"$"` expression.

Example: `["$", "MIN_AGE"]`

As shorthand, a parameter expressions can be collapsed into a one-element array, like `["$MIN_AGE"]`.

### Variables

Variables are placeholders used in a `ANY` and `EVERY` expression to represent the collection item being iterated over. The (required) first operand of the `"?"` expression is the variable's name, and the (optional) extra operands are a property path relative to the variable's value.

Example: `["?", "address", "zip"]`

As shorthand, a variable expression can be collapsed into a one-element array, like `["?address"]` or `["?address.zip"]`.

## Operations

The operations are named after their N1QL/SQL equivalents.

|Category| Name | Operand Count |
|--------|------|---------------|
|Arithmetic| `+`  | 2+ |
| | `-`  | 1 or 2 |
| | `*`  | 2+ |
| | `/`  | 2 |
| | `%`  | 2 |
|String| `||`  | 2+ |
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
| | `IS MISSING` | 1 |
| | `IS NOT MISSING` | 1 |
| | `IS NULL` | 1 |
| | `IS NOT NULL` | 1 |
| | `COLLATE` | 2: (options, expr) [see **[Collation](#collation)** below] |
|Logical| `NOT` | 1 |
| | `AND` | 2+ |
| | `OR` | 2+ |
|Functions| _name_`()` | Depends on [function](#functions) |
|Conditional| `CASE` | 2+: (expr, when1, ...) |
| | `WHEN` | 2: (cond, value) |
| | `ELSE` | 1: (value) |
|Collections| `ANY` | 3: (variable name, array, satisfies) |
| | `EVERY` | 3: (variable name, array, satisfies) |
| | `ANY AND EVERY` | 3: (variable name, array, satisfies) |
|Properties| `.` | 0+: (path components) [[see above](#properties)] |
|Parameters| `$` | 1 (name or position) [[see above](#parameters)] |
|Variables| `?` | 1+ (name, optional path components) [[see above](#variables)] |
|Queries| `SELECT` | 1 [[see below](#top-level-query-and-select)] |

## Collation

The `COLLATE` operator does nothing itself, merely returns the value of its second operand, but it alters the string collation (comparison/sorting) used when evaluating that expression _and nested expressions_. The first operand is a dictionary that specifies the collation; its keys are:

| Key | Value | Default Value |
|-----|-------|---------------|
| `UNICODE` | Unicode-aware? | `false` |
| `CASE` | Case-sensitive? | `true` |
| `DIAC` | Diacritic (accent) -sensitive? | `true` |
| `LOCALE` | ISO locale* string or `null` | `null` |

\* A **locale** is an ISO-639 [language code](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes) plus, optionally, an underscore and an ISO-3166 [country code](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2): `"en"`, `"en_US"`, `"fr_CA"`, etc.

Some details on combining keys:

* If `UNICODE` is not true, only the `CASE` value is significant: a case-sensitive collation is a purely binary string comparison; a case-insensitive one also treats ASCII uppercase and lowercase letters as equivalent.
* If `UNICODE` is true, but `LOCALE` is missing or null, the collation is Unicode-aware but not localized; for example, accented Roman letters sort right after the base letter. (This is implemented by using the "en_US" locale.)
* Any keys not specified are inherited from the enclosing context.
* There's implicitly a top-level context with the default values for the keys, i.e. `{UNICODE: false, CASE: true, DIAC: true, LOCALE: null}`.

### About Unicode Collation

The details of [Unicode collation](http://userguide.icu-project.org/collation) are quite complex, though for the most part it just Does The Right Thing according to a human of that locale. But it doesn't behave like the simple `strcmp` and `strcasecmp` functions that programmers are used to!

* The collation algorithm first compares the strings ignoring case and diacritics, just looking at the base letters. If the letters are not equal, it stops and returns the relative ordering based on the mismatched letters. This is usually, but _not always_, the ordering English speakers are used to; for example, in Lithuanian "Y" comes after "I", and in traditional Spanish the sequence "CH" sorts as a single letter that comes between "C" and "D".
* Otherwise, if diacritic-sensitive, it compares the strings again, this time considering also diacritics (accents). If they differ, it returns the relative ordering. By default an accented letter sorts just after the base letter, but many locales have special rules: "Å" in Danish is treated as a separate letter that sorts just after "Z".
* Otherwise, if case-sensitive, it compares the strings again, now considering case; if they differ in case, it returns the relative ordering, with lowercase coming _before_ uppercase (contrary to the intuition of programmers used to ASCII ordering!)

The above applies to the Roman alphabet. For many non-Roman scripts, especially Chinese and Japanese, the rules get much more complicated.

Some unintuitive results: in a case-sensitive collation, "abc" comes before "ABC" (lowercase first!), but "abd" comes _after_ "ABC" because the letter mismatch takes priority over the case mismatch. Likewise, "ápple" comes after "Apple" (in most locales) because the diacritic is higher priority than uppercase.

## Top-Level Query, and `SELECT`

The `SELECT` statement has so many parameters, all of which are optional, that it makes a lot more sense to encode them as a dictionary with the following keys:

| Key | Value | Default Value |
|-----|-------|---------------|
| `WHAT` | Array of expressions to return, generally properties | document ID and sequence |
| `FROM` | Array of [database/join identifiers](#databasejoin-identifiers) | Database being queried |
| `WHERE` | Boolean-valued expression | `true` (all documents) |
| `HAVING` | Expression | `true` |
| `DISTINCT` | Boolean | `false` |
| `GROUP_BY` | Array of expressions or property names | `[]` (no grouping) |
| `ORDER_BY` | Array of expressions or property names | `[]` (unsorted) |
| `LIMIT` | Number | Infinite |
| `OFFSET` | Number | 0 |

### Database/Join Identifiers

The items in the `FROM` array are dictionaries with the following keys:

| Key | Value | Default Value |
|-----|-------|---------------|
| `AS` | Alphanumeric string: an alias to refer to this database or join by | _required_ |
| `DB` | String: Database name | Database being queried |
| `JOIN` | String: Type of join | `"INNER"` (if `ON` is given) |
| `ON` | Boolean-valued expression: the join constraint | no join |

Some requirements:
* The first item in the array serves only to alias the default database; it can't have a `JOIN` or `ON` property.
* The subsequent items _must_ be joins, with `ON` properties.
* It's an error to have a `JOIN` property but not an `ON`.
* Legal values for `JOIN` are `"INNER"`, `"OUTER"`, `"LEFT OUTER"`, and `"CROSS"`. (Case-insensitive)
* All `AS` values must be unique.

**STATUS:** (March 2017) The `DB`property is not yet implemented; only one database can be queried at a time.

Example:
```json
    "FROM": [{"as": "person"},
             {"as": "state", "on": ["=", [".state.abbreviation"],
                                         [".person.contact.address.state"]]}],
```

**Adding a `FROM` clause affects the interpretation of properties.** Since there are usually multiple databases or join sources, property names (paths) in the entire query need to be disambiguated by prefixing the appropriate alias. So in the above example, a document's `abbreviation` property has to be named as `.state.abbreviation`, not just `.abbreviation`.

## Functions

These are N1QL functions. For detailed information about parameters and results, please consult the [N1QL documentation](https://developer.couchbase.com/documentation/server/4.5/n1ql/n1ql-language-reference/functions.html). 

**NOTE:** There are some differences from SQL, or at least from SQLite; for example, SQLite has non-aggregate versions of `min` and `max`, but in N1QL (and LiteCore) these are called `least` and `greatest`.

|Category| Name | Operand Count |
|--------|------|---------------|
| **Aggregate** | `avg()` | 1 |
| | `count()` | 1 |
| | `max()` | 1 |
| | `min()` | 1 |
| | `sum()` | 1 |
| **Arrays** | `array_avg()` | 1 |
| | `array_contains()` | 2 |
| | `array_count()` | 1 |
| | `array_ifnull()` | 1 |
| | `array_length()` | 1 |
| | `array_max()` | 1 |
| | `array_min()` | 1 |
| | `array_sum()` | 1 |
| **Comparisons** | `greatest()` | 2+ |
| | `least()` | 2+ |
| **Conditionals** | `ifmissing()` | 2+ |
| | `ifmissingornull()` | 1+ |
| | `ifnull()` | 1+ |
| | `missingif()` | 2 |
| | `nullif()` | 2 |
| **Math** | `abs()` | 1 |
| | `acos()` | 1 |
| | `asin()` | 1 |
| | `atan()` | 1 |
| | `atan2()` | 2 |
| | `ceil()` | 1 |
| | `cos()` | 1 |
| | `degrees()` | 1 |
| | `e()` | 0 |
| | `exp()` | 1 |
| | `ln()` | 1 |
| | `log()` | 1 |
| | `floor()` | 1 |
| | `pi()` | 0 |
| | `power()` | 2 |
| | `radians()` | 1 |
| | `round()` | 1–2 |
| | `sign()` | 1 |
| | `sin()` | 1 |
| | `sqrt()` | 1 |
| | `tan()` | 1 |
| | `trunc()` | 1–2 |
| **Patterns** | `regexp_contains()` | 2 |
| | `regexp_like()` | 2 |
| | `regexp_position()` | 2 |
| | `regexp_replace()` | 3-4 |
| | `rank()` | 1 |
| **Strings** | `contains()` | 2 |
| | `length()` | 1 |
| | `lower()` | 1 |
| | `ltrim()` | 1–2 |
| | `rtrim()` | 1–2 |
| | `trim()` | 1–2 |
| | `upper()` | 1 |
| **Types** | `isarray()` | 1 |
| | `isatom()` | 1 |
| | `isboolean()` | 1 |
| | `isnumber()` | 1 |
| | `isobject()` | 1 |
| | `isstring()` | 1 |
| | `type()` | 1 |
| | `toarray()` | 1 |
| | `toatom()` | 1 |
| | `toboolean()` | 1 |
| | `tonumber()` | 1 |
| | `toobject()` | 1 |
| | `tostring()` | 1 |

## Implementation Status

(Updated August 18, 2017)

Still TBD, probably coming post-2.0:

* Multi-database JOINs
* Left, outer, cross, natural, ingrown, dovetail JOINs
* More N1QL functions; many of the remaining ones are useless in a SELECT query (like `random`), but there are a number of array and object operations that can be useful.