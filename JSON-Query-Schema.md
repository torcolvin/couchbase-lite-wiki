## Table Of Contents

* [Introduction](#introduction)
* [Example](#example)
* [Leaf Types](#leaf-types)
* [Operations](#operations)
* [Collation](#collation)
* [Top-Level Query, and `SELECT`](#top-level-query-and-select)
* [Functions](#functions)
* [Implementation Status](#implementation-status)

## Introduction

Queries are expressed to LiteCore as JSON, so they can be easily transformed and converted to internal representations like [SQL](http://www.sqlite.org/lang_expr.html). This document describes the schema.

The JSON describes a parse tree. Each node of the tree describes an operation and a list of operands (children). The operations can be arithmetic, comparison, logical, etc. The number of children depends on the operation; for example, `NOT` has exactly one, `-` has one or two (negation or subtraction), `AND` has two or more.

A typical way to represent a parse tree is as nested lists or arrays, where the first element represents the operator and the rest represent the operands; for example `["=", ["+", 2, 2], 5]`.

In addition to the operators from SQL and N1QL, we need ones to represent document property paths and query parameters. Operator `"."` represents a path, and `"$"` a parameter.

**NOTE:** This schema is case-insensitive, like SQL and N1QL. All operation names, function names, and `SELECT` keys can be upper- or lower-case or any mixture.

## Example

`SELECT name.first, name.last FROM students WHERE grade = 12 AND gpa >= $GPA`

As a JSON tree this looks like:

```
["SELECT", {
    "WHAT": [
        [".", "name", "first"],
        [".", "name", "last"] ],
    "FROM":
        "students",
    "WHERE":
        ["AND",
            ["=",
                [".", "grade"],
                12],
            [">=",
                [".", "gpa"],
                ["$", "GPA"] ] } ]
```

## Leaf Types

| Type | Representation | Example |
|------|----------------|---------|
| Constant | JSON scalar | `true`, `null`, `17`, `"foo"` |
| Property | `.` operation | `[".", "name", "first"]` |
| Parameter | `$` operation | `["$", "MIN_AGE"]` |
| Variable | `?` operation | `["?", "Item"]` `["?", "Item", "price"]` |

As shorthand, properties/parameters/variables can be collapsed into one-element arrays, like `[".name.first"]` and `["$MIN_AGE"]`.

Note: The special property names `_id` and `_sequence` refer to the document's ID and current sequence number.

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
| | `IN` | 2+: (value, option1, ...) |
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
|Properties| `.` | 1+: (path components) |
|Parameters| `$` | 1 (name or position) |
|Variables| `?` | 1+ (name, optional path components) |
|Queries| `SELECT` | 1 [[see below](#top-level-query-and-select)] |

## Collation

The `COLLATE` operator does nothing itself, merely returns the value of its second parameter, but it alters the string collation used when evaluating that expression and nested expressions. The first parameter is a dictionary that specifies the collation; its keys are:

| Key | Value | Default Value |
|-----|-------|---------------|
| `UNICODE` | Unicode-aware? | `false` |
| `CASE` | Case-sensitive? | `true` |
| `DIAC` | Diacritic (accent) -sensitive? | `true` |
| `LOCALE` | ISO locale or language code (`"en"`, `"en_US"`, etc.) | system language |

* If `UNICODE` is not true, `DIAC` and `LOCALE` are ignored.
* If `UNICODE` is true, but `LOCALE` is missing or null, the collation is Unicode-aware but not localized; for example, accented Roman letters sort right after the base letter.
* Any keys not specified are inherited from the enclosing context.
* There's implicitly a top-level context with the default values for the keys, i.e. `{UNICODE: false, CASE: true, DIAC: true, LANG: $system_language}`.

**STATUS:** (July 2017) Still under design; implementation coming ASAP

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

**STATUS:** (Feb 2017) _Most of these functions are unimplemented!_ For now, the rule of thumb is that, if it's not a built-in [SQLite  function](http://www.sqlite.org/lang_corefunc.html) or [aggregate](http://www.sqlite.org/lang_aggfunc.html), it won't work.

|Category| Name | Operand Count |
|--------|------|---------------|
| **Aggregate** | `array_agg()` | |
| | `avg()` | |
| | `count()` | |
| | `max()` | |
| | `min()` | |
| | `sum()` | |
| **Arrays** | `array_append()` | |
| | `array_avg()` | |
| | `array_concat()` | |
| | `array_contains()` | |
| | `array_count()` | |
| | `array_distinct()` | |
| | `array_ifnull()` | |
| | `array_insert()` | |
| | `array_intersect()` | |
| | `array_length()` | |
| | `array_max()` | |
| | `array_min()` | |
| | `array_position()` | |
| | `array_prepend()` | |
| | `array_put()` | |
| | `array_range()` | |
| | `array_remove()` | |
| | `array_repeat()` | |
| | `array_replace()` | |
| | `array_reverse()` | |
| | `array_sort()` | |
| | `array_sum()` | |
| | `array_star()` | |
| **Comparisons** | `greatest()` | |
| | `least()` | |
| **Conditionals** | `ifinf()` | |
| | `ifmissing()` | |
| | `ifmissingornull()` | |
| | `ifnan()` | |
| | `ifnanorinf()` | |
| | `ifnull()` | |
| | `missingif()` | |
| | `nanif()` | |
| | `nullif()` | |
| | `neginfif()` | |
| | `posinfif()` | |
| **Math** | `abs()` | |
| | `acos()` | |
| | `asin()` | |
| | `atan()` | |
| | `atan2()` | |
| | `ceil()` | |
| | `cos()` | |
| | `degrees()` | |
| | `e()` | |
| | `exp()` | |
| | `ln()` | |
| | `log()` | |
| | `floor()` | |
| | `pi()` | |
| | `power()` | |
| | `radians()` | |
| | `random()` | |
| | `round()` | |
| | `sign()` | |
| | `sin()` | |
| | `sqrt()` | |
| | `tan()` | |
| | `trunc()` | |
| **Misc** | `base64()` | |
| | `base64_encode()` | |
| | `base64_decode()` | |
| | `uuid()` | |
| **Objects** | `object_length()` | |
| | `object_names()` | |
| | `object_pairs()` | |
| | `object_length()` | |
| | `object_inner_pairs()` | |
| | `object_values()` | |
| | `object_inner_values()` | |
| | `object_add()` | |
| | `object_put()` | |
| | `object_remove()` | |
| | `object_unwrap()` | |
| **Patterns** | `regexp_contains()` | |
| | `regexp_like()` | |
| | `regexp_position()` | |
| | `regexp_replace()` | |
| | `rank()` | |
| **Strings** | `contains()` | |
| | `initcap()` | |
| | `length()` | |
| | `lower()` | |
| | `ltrim()` | |
| | `position()` | |
| | `repeat()` | |
| | `replace()` | |
| | `rtrim()` | |
| | `split()` | |
| | `substr()` | |
| | `suffixes()` | |
| | `title()` | |
| | `trim()` | |
| | `upper()` | |
| **Types** | `isarray()` | |
| | `isatom()` | |
| | `isboolean()` | |
| | `isnumber()` | |
| | `isobject()` | |
| | `isstring()` | |
| | `type()` | |
| | `toarray()` | |
| | `toatom()` | |
| | `toboolean()` | |
| | `tonumber()` | |
| | `toobject()` | |
| | `tostring()` | |

## Implementation Status

(Updated July 31, 2017)

### Phase 2: Nested Operators and MISSING

* Implemented ANY / EVERY operators.
* `NULL` (which is a JSON null, not a SQL null!) is represented by a zero-length blob in the generated SQL.

### Phase 3: Projection

* Implemented `WHAT` property of the `SELECT` object.

### Phase 4: Aggregate functions and Group By

* Implemented `GROUP_BY` and `DISTINCT` properties of the `SELECT` object
* Implemented support for aggregate functions

### Phase 5: Joins

* Basic inner joins implemented.
* No support for multiple databases yet.
* No support for left, outer, cross, natural, ingrown or dovetail joins.

### Phase 6: Collation

* Unicode-aware collation functions being implemented in LiteCore
* Specifying JSON schema of collation specifiers
* Collation specifiers not recognized by query parser yet