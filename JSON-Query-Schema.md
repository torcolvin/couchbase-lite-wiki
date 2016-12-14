Queries are expressed to LiteCore as JSON, so they can be easily transformed and converted to internal representations like SQL. This document describes the schema.

The JSON describes a parse tree. Each node of the tree describes an operation and a list of operands (children). The operations can be arithmetic, comparison, logical, etc. The number of children depends on the operation; `NOT` has exactly one, `-` has one or two (negation or subtraction), `AND` has two or more.

A nice compact way to represent this is as a one-item JSON object whose key represents the operation and value represents the operands; for example `{"AND": [{...}, {...}]}`. If there's only one operand we don't need to put it in an array.

## Values

The leaves of the tree are values like constants, property names and query parameter names. We can represent constants as their equivalent JSON scalar values, and the others as special operands whose values are strings.

| Type | Representation | Example |
|------|----------------|---------|
| Constant | JSON scalar | `true`, `null`, `17`, `"foo"` |
| Property | `prop` operation | `{"prop": "name.first"}` |
| Parameter | `param` operation | `{"param": "MIN_AGE"}` |

## Operations

The operations can be named after their N1QL/SQL equivalents.

| Name | Operand Count |
|------|---------------|
| **Arithmetic:** |
| `+`  | 2+ |
| `-`  | 1 or 2 |
| `*`  | 2+ |
| `/`  | 2 |
| `%`  | 2 |
| `||`  | 2+ |
| **Relational:** |
| `=` | 2 |
| `!=` | 2 |
| `<` | 2 |
| `<=` | 2 |
| `>` | 2 |
| `>=` | 2 |
| `BETWEEN` | 3 (value, min, max) |
| `IS` | 2 |
| `IS NOT` | 2 |
| `LIKE` | 2 |
| `IN` | 2+ (value, option1, ...) |
| `Exists` | 1 |
| **Logical:** |
| `NOT` | 1 |
| `AND` | 2+ |
| `OR` | 2+ |
| **Functions:** |
| _name_`()` | Depends on function |
| **Conditional:** |
| `CASE` | 1+ |
| **Nested:** |
| `.` | 2 (parent, child) |
| `[]` | 1+ (index or range) |
| **Collection:** |
| `ANY` | 1 (expression) |
| `EVERY` | 2 (expression) |
| `ANY AND EVERY` | 2 (expression) |
| **Properties:** |
| `prop` | 1 (path string) |
| **Parameters:** |
| `param` | 1 (name or position) |
| **Query** |
| `QUERY` | 1+ (of types below) |
| `SELECT` | 1+ |
| `FROM` | 1+ (database names) |
| `WHERE` | 1 |
| `ORDER` | 1+ |
| `LIMIT` | 1 |
| `OFFSET` | 1 |

## Example

`SELECT name.first, name.last FROM students WHERE grade = 12 AND gpa >= 4.0`

As a JSON tree this looks like:

```
{QUERY: [
  {SELECT: [
    {prop: "name.first"},
    {prop: "name.last"} ] },
  {FROM:
    "students"},
  {WHERE:
    {AND: [
      {"=": [
        {prop: "grade"},
        12 ]},
      {">=": [
        {prop: "gpa"},
        4.0 ]} ]}} ]}
```

##Phase 2

Nested and Collection Operators

### Rules
- ANY: If ANY entry (address) in the array (addresses) matches the expression return TRUE, otherwise FALSE. Return FALSE for an array w/ no entries.
- EVERY: If EVERY entry (address) in the array (addresses) matches the expression return TRUE, otherwise FALSE. Return TRUE for an array w/ no entries.
- ANY AND EVERY: If EVERY entry (address) in the array (addresses) matches the expression return TRUE, otherwise FALSE. Return FALSE for an array w/ no entries.


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
