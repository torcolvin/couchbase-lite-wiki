Queries are expressed to LiteCore as JSON, so they can be easily transformed and converted to internal representations like [SQL](http://www.sqlite.org/lang_expr.html). This document describes the schema.

The JSON describes a parse tree. Each node of the tree describes an operation and a list of operands (children). The operations can be arithmetic, comparison, logical, etc. The number of children depends on the operation; for example, `NOT` has exactly one, `-` has one or two (negation or subtraction), `AND` has two or more.

A typical way to represent a parse tree is as nested lists or arrays, where the first element represents the operator and the rest represent the operands; for example `["=", ["+", 2, 2], 5]`.

In addition to the operators from SQL and N1QL, we'll need ones to represent document property paths and query parameters. We'll use operator `"."` for paths, and `"$"` for parameters.

## Leaf Types

| Type | Representation | Example |
|------|----------------|---------|
| Constant | JSON scalar | `true`, `null`, `17`, `"foo"` |
| Property | `.` operation | `[".", "name", "first"]` |
| Parameter | `$` operation | `["$", "MIN_AGE"]` |

Note: The special property names `_id` and `_sequence` refer to the document's ID and current sequence number.

## Operations

The operations can be named after their N1QL/SQL equivalents.

|Category| Name | Operand Count |
|--------|------|---------------|
|Arithmetic| `+`  | 2+ |
| | `-`  | 1 or 2 |
| | `*`  | 2+ |
| | `/`  | 2 |
| | `%`  | 2 |
| | `||`  | 2+ |
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
|Logical| `NOT` | 1 |
| | `AND` | 2+ |
| | `OR` | 2+ |
|Functions| _name_`()` | Depends on function |
|Conditional| `CASE` | 2+: (expr, when1, ...) |
| | `WHEN` | 2: (cond, value) |
| | `ELSE` | 1: (value) |
|Collections| `ANY` | 1: (expression) |
| | `EVERY` | 2: (expression) |
| | `ANY AND EVERY` | :2 (expression) |
|Properties| `prop` | 1+: (path components) |
|Parameters| `param` | 1 (name or position) |
|Queries| `SELECT` | 1 [see below] |

## Top-Level Query

The `SELECT` statement has so many parameters, all of which are optional, that it makes a lot more sense to encode them as a dictionary, with keys `WHAT`, `FROM`, `WHERE`, `ORDER BY`, `LIMIT`, `OFFSET`.

| Key | Value | Default Value |
|-----|-------|---------------|
| `WHAT` | Array of expressions to return, generally properties | Entire document |
| `FROM` | Array of database identifiers (format TBD) | Database being queried |
| `WHERE` | Boolean-valued expression | Always true (all documents) |
| `ORDER BY` | Expression(s) | Document ID (`_id`) |
| `LIMIT` | Number | Infinite |
| `OFFSET` | Number | 0 |


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

## Phase 2: Nested Operators and MISSING

### Rules

* ANY: If ANY entry (address) in the array (addresses) matches the expression return TRUE, otherwise FALSE. Return FALSE for an array w/ no entries.
* EVERY: If EVERY entry (address) in the array (addresses) matches the expression return TRUE, otherwise FALSE. Return TRUE for an array w/ no entries.
* ANY AND EVERY: If EVERY entry (address) in the array (addresses) matches the expression return TRUE, otherwise FALSE. Return FALSE for an array w/ no entries.

### Examples

```
SELECT *
FROM contacts
WHERE type = "contact"
AND (social.twitter NOT NULL OR social.github NOT NULL)
AND length(firstName) > 0
AND ANY address IN addresses SATISFIES address.country IS NOT MISSING END
AND EVERY address IN addresses SATISFIES address.street1 IS NOT MISSING END
AND ANY AND EVERY address IN addresses SATISFIES address.city IS NOT MISSING END
```

##Phase 3
Joins
```
SELECT * from `contact` contact  JOIN 'contact' order 
where  contact.user_id = order.requestorID 
```
##Phase 4 
Projection
```
SELECT contact.firstName, contact.lastName from `contact` contact  JOIN 'contact' order 
where  contact.user_id = order.requestorID 
```

## Phase 5
###Full Text

###Index

Will have support for full and partial indexes but won't support indexing of values in Arrays. 

```
CREATE INDEX over5 ON `beer-sample`(abv) WHERE abv > 5
```