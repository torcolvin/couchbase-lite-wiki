Queries are expressed to LiteCore as JSON, so they can be easily transformed and converted to internal representations like [SQL](http://www.sqlite.org/lang_expr.html). This document describes the schema.

The JSON describes a parse tree. Each node of the tree describes an operation and a list of operands (children). The operations can be arithmetic, comparison, logical, etc. The number of children depends on the operation; for example, `NOT` has exactly one, `-` has one or two (negation or subtraction), `AND` has two or more.

A typical way to represent a parse tree is as nested lists or arrays, where the first element represents the operator and the rest represent the operands; for example `["=", ["+", 2, 2], 5]`.

In addition to the operators from SQL and N1QL, we'll need ones to represent document property paths and query parameters. We'll use operator `"."` for paths, and `"$"` for parameters.

**NOTE:** This schema is case-insensitive, like SQL and N1QL. All operation names, function names, and `SELECT` keys can be upper- or lower-case or any mixture.

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

The operations can be named after their N1QL/SQL equivalents.

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
|Logical| `NOT` | 1 |
| | `AND` | 2+ |
| | `OR` | 2+ |
|Functions| _name_`()` | Depends on function |
|Conditional| `CASE` | 2+: (expr, when1, ...) |
| | `WHEN` | 2: (cond, value) |
| | `ELSE` | 1: (value) |
|Collections| `ANY` | 3: (variable name, array, satisfies) |
| | `EVERY` | 3: (variable name, array, satisfies) |
| | `ANY AND EVERY` | 3: (variable name, array, satisfies) |
|Properties| `.` | 1+: (path components) |
|Parameters| `$` | 1 (name or position) |
|Variables| `?` | 1+ (name, optional path components) |
|Queries| `SELECT` | 1 [see below] |

## Top-Level Query, and `SELECT`

The `SELECT` statement has so many parameters, all of which are optional, that it makes a lot more sense to encode them as a dictionary with the following keys, all optional:

| Key | Value | Default Value |
|-----|-------|---------------|
| `WHAT` | Array of expressions to return, generally properties | document ID and sequence |
| `FROM` | Array of database identifiers (format TBD) | Database being queried |
| `WHERE` | Boolean-valued expression | `true` (all documents) |
| `HAVING` | Expression | `true` |
| `DISTINCT` | Boolean | `false` |
| `GROUP_BY` | Array of expressions or property names | `[]` (no grouping) |
| `ORDER_BY` | Array of expressions or property names | `[]` (unsorted) |
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

## Implementation Status

### Phase 2: Nested Operators and MISSING

* Implemented ANY / EVERY operators.
* Thinking about how to distinguish MISSING from NULL in the generated SQL.

### Phase 3: Projection

* Implemented `WHAT` property of the `SELECT` object.

### Phase 4: Aggregate functions and Group By

* Implemented `GROUP_BY` and `DISTINCT` properties of the `SELECT` object
* Implemented support for aggregate functions

### Phase 5: Joins

```
SELECT * FROM `contact` as contact  JOIN 'contact' as order 
WHERE  contact.user_id = order.requestorID 
```