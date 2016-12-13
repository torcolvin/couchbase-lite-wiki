Queries are expressed to LiteCore as JSON, so they can be easily transformed and converted to internal representations like SQL. This document describes the schema.

The JSON describes a parse tree. Each node of the tree describes an operation and a list of operands (children). The operations can be arithmetic, comparison, logical, etc. The number of children depends on the operation; `NOT` has exactly one, `-` has one or two (negation or subtraction), `AND` has two or more.

A nice compact way to represent this is as a one-item JSON object whose key represents the operation and value represents the operands; for example `{"AND": [{...}, {...}]}`. If there's only one operand we don't need to put it in an array.

The leaves of the tree are things like constants, property names and query parameter names. We can represent constants as their equivalent JSON scalar values, and the others as special operands whose values are strings.

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
| **Logical:** |
| `NOT` | 1 |
| `AND` | 2+ |
| `OR` | 2+ |
| **Functions:** |
| _name_`()` | Depends on function |
| **Properties:** |
| `prop` | 1 (path string) |
| **Parameters:** |
| `param` | 1 (name or position) |
