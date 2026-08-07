# JSON Encoding

This document describes the compact JSON encoding format defined in the SOIL core libraries.

## Structure

The compact JSON format is structurally the same as the [JSON Encoding](JSON.md).
It only differs in the field names, which are single characters for the compact format.
Additionally, the message and value types are encoded as single-digit integers, not as strings.

### Field names

The following table explains how the regular JSON field names are translated to the compact representation:

| JSON Field    | Compact name | Comment                                                           |
|:--------------|:-------------|:------------------------------------------------------------------|
| `children`    | `n`          | `n` is derived from `nodes`, since `c` is already used by `count` |
| `count`       | `m`          |                                                                   |
| `depth`       | `d`          |                                                                   |
| `end`         | `e`          |                                                                   |
| `id`          | `m`          |                                                                   |
| `message`     | `m`          |                                                                   |
| `path`        | `p`          |                                                                   |
| `samples`     | `m`          |                                                                   |
| `time`        | `m`          |                                                                   |
| `type`        | `h`          | `h` is derived from `hint`, since `t` is already used by `time`   |
| `value`       | `v`          |                                                                   |

### Message type

The message type is transmitted as an integer, according to the following table:

| JSON type | Compact value |
|:----------|:--------------|
| `data`    | `0`           |
| `read`    | `1`           |
| `delete`  | `2`           |
| `include` | `3`           |
| `exclude` | `4`           |
| `clear`   | `5`           |

### Value type

The value type is also transmitted as an integer:

| SOIL Type | JSON Type   | Compact value |
|:----------|:------------|:--------------|
| Null      | `null`      | `0`           |
| Integer   | `"integer"` | `1`           |
| Double    | `"double"`  | `2`           |
| String    | `"string"`  | `3`           |
| Data      | `"data"`    | `4`           |
| Path      | `"path"`    | `5`           |
