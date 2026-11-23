# JSON Encoding

This document describes the binary encoding format defined in the SOIL core libraries.

### Table of contents

- [Message types](#message-types)
- [Data messages](#data-messages)
- [Request messages](#request-messages)
- [Deletion requests](#data-deletion-messages)
- [Subscription inclusion](#subscription-inclusion-messages)
- [Subscription exclusion](#subscription-exclusion-messages)
- [Subscription deletion](#subscription-deletion-messages)
- [Examples](#examples)

## Message types

There are currently six different messages in SOIL:

| Type                                                       | Raw Value | Usage                                       |
|:-----------------------------------------------------------|:----------|:--------------------------------------------|
| [Data message](#data-messages)                             |       `0` | Transmit a tree for a path                  |
| [Request message](#request-messages)                       |       `1` | Request a tree, including history           |
| [Deletion message](#data-deletion-messages)                |       `2` | Delete parts of a tree, including history   |
| [Subscription inclusion](#subscription-inclusion-messages) |       `3` | Request future updates for a path or tree   |
| [Subscription exclusion](#subscription-exclusion-messages) |       `4` | Prevent future updates for a path or tree   |
| [Subscription deletion](#subscription-deletion-messages)   |       `5` | Clear subscription rules for a path or tree |

The message type is encoded as the first three bits (MSB) of any message.

## Data messages

#### Structure

`TYPE | HAS_ID | HAS_COUNT | RESERVED | [ID] | [COUNT] | TREE`

| Field           | Start bit | Length | Content                                           |
|:----------------|:----------|:-------|:--------------------------------------------------|
| `TYPE`          | 0         | 3      | `0` (see [message type](#message-types))          |
| `HAS_ID`        | 3         | 1      | `1` if the header contains an `ID` field          |
| `HAS_COUNT`     | 4         | 1      | `1` if the header contains a `COUNT` field        |
| `RESERVED`      | 5         | 3      | `0b000` Unused bits                               |
| `ID`            | 8         | 0 - 72 | The message `ID`, encoded as [VarInt](#varint)    |
| `COUNT`         | 8-80      | 0 - 72 | The message `COUNT`, encoded as [VarInt](#varint) |
| `TREE`          | 8-152     | ?      | The data being transmitted, see [tree](#tree)     |

> For a simple data message (no id, no count) the header byte is just `0x00`.

### Tree

#### Structure

`CHILD_COUNT | SAMPLE_COUNT | [EXTENDED_CHILD_COUNT] | [EXTENDED_SAMPLE_COUNT] | [SAMPLES] | [CHILD_NODES]`

#### Fields

| Field                       | Start bit | Length | Content                                                                    |
|:----------------------------|:----------|:-------|:---------------------------------------------------------------------------|
| `CHILD_COUNT`               | 1         | 3      | The number of child nodes in this node, clamped to `0b1111`                |
| `SAMPLE_COUNT`              | 5         | 3      | The number of samples for this node, clamped to `0b1111`                   |
| `EXTENDED_CHILD_COUNT`      | 8         | 0-72   | A [VarInt](#varint) with the total child count, if `CHILD_COUNT = 0b1111`  |
| `EXTENDED_SAMPLE_COUNT`     | 8-80      | 0-72   | A [VarInt](#varint) with the total sample count, if `CHILD_COUNT = 0b1111` |
| `SAMPLES`                   | 8-152     | ?      | The samples of this node, see [sample](#sample)                            |
| `CHILD_NODES`               | ?         | ?      | The children of this node, see [child](#child)                             |

The `EXTENDED_CHILD_COUNT` and `EXTENDED_SAMPLE_COUNT` are only needed for tree nodes that have more than 14 samples/children.
It will thus rarely be needed, shrinking the header in most cases to a single byte per node.

> For a simple message with only one sample, the tree simplifies to: `0x01 | SAMPLE`

### Child

#### Structure

`NAME | TREE`

#### Fields

| Field  | Start bit | Length | Content                                            |
|:-------|:----------|:-------|:---------------------------------------------------|
| `NAME` | 0         | 8-72   | The name of the child, encoded as [STRING](#value) |
| `TREE` | ?         | ?      | The data of the child, see [tree](#tree)           |

### Sample

#### Structure

`HAS_TIMESTAMP | TYPE | [TIMESTAMP] | [VALUE]`

#### Fields

| Field           | Start bit | Length | Content                                                                  |
|:----------------|:----------|:-------|:-------------------------------------------------------------------------|
| `HAS_TIMESTAMP` | 0         | 1      | `1` if the data contains a `TIMESTAMP` field                             |
| `TYPE`          | 1         | 7      | The type of the value, see [value](#value)                               |
| `TIMESTAMP`     | 8         | 64     | The timestamp of the sample, encoded as [`TIMESTAMP`](#timestamp)        |
| `VALUE`         | 8/72      | ?      | The value of the sample (Optional if `TYPE = NULL`), see [value](#value) |

> For a sample without a timestamp, the first byte simply denotes the `TYPE`, e.g. `0x01` for an integer.

### Value

The `VALUE` field is only used for [samples](#sample) in [data messages](#data-message).

| SOIL Type | [`VALUE_TYPE`](#value-type) | Encoding                                                           |
|:----------|:----------------------------|:-------------------------------------------------------------------|
| Null      | `0`                         | No data                                                            |
| Integer   | `1`                         | [ZigZag VarInt](#zigzag-varint)                                    |
| Double    | `2`                         | 64-bit IEEE double, big endian                                     |
| String    | `3`                         | length as [VarInt](#varint) + UTF-8                                |
| Data      | `4`                         | length as [VarInt](#varint) + bytes                                |
| Path      | `5`                         | count as [VarInt](#varint) + components as `STRING`                |
| List      | `6`                         | count as [VarInt](#varint) + components as `TYPE + VALUE`          |
| Map       | `7`                         | count as [VarInt](#varint) + components as `STRING + TYPE + VALUE` |

Examples:

| Value                 | Encoding (excluding `VALUE_TYPE`) |
|:----------------------|:----------------------------------|
| Null                  | *No data*                         |
| `123` (Integer)       | `F6 01`                           |
| `3.14`(Double)        | `40 09 1E B8 51 EB 85 1F`         |
| `"abc"` (String)      | `03 61 62 63`                     |
| `0xABBA` (Data)       | `02 AB BA`                        |
| `"a", "b"` (Path)     | `02 01 61 01 62`                  |
| `["a", 123]` (List)   | `02 03 01 61 01 F6 01`            |
| `{ "a" : 123 }` (Map) | `01 03 01 61 01 F6 01`            |

### Timestamp

A timestamp defined as a double value of the UTC seconds since January 1st, 1970.
It is encoded as a [DOUBLE](#value).

### Varint

Varints are variable-length integer encodings, used in other binary protocols, such as in [Google protocol buffers](https://protobuf.dev/programming-guides/encoding/).
The first bit (MSB) of every byte (except the last) indicates if there is an additional byte following the current one,
thereby encoding a 64-bit integer using between 1 and 9 bytes.

Values between 0 and 127 are encoded using a single byte:

| Value | Encoding     |
|:------|:-------------|
| `0`   | `0b00000000` |
| `1`   | `0b00000001` |
| `1`   | `0b00000010` |
| ...   | ...          |
| `126` | `0b01111110` |
| `127` | `0b01111111` |

For larger values, the first bit (MSB) indicates that more data follows, up to 9 bytes total.
The last byte has no indicator bit, and all bits are used.
To compute the integer value:

```
value = 0
repeat 8 times:
    byte = nextByte()
    dataBits = byte & 0b01111111
    value = (value << 7) + dataBits
    if byte & 0b10000000:
        return value
byte = nextByte()
value = (value << 8) + byte
return value
```

 > Note: While the encoding/decoding is similar to Google Protocol Buffers, it is not identical.
 > Here, the last byte has no continuation bit, and all 8 bits are used.

For example, the number `150` is encoded as `0x9601`, or `10010110 00000001`:

```
10010110 00000001        // Original inputs.
 0010110  0000001        // Drop continuation bits.
 0000001  0010110        // Convert to big-endian.
   00000010010110        // Concatenate.
 128 + 16 + 4 + 2 = 150  // Interpret as an unsigned 64-bit integer.
```

### ZigZag VarInt

Signed integer values are encoded as [VarInt](#varint), except that positive integers `p` are encoded as `2 * p` (the even numbers), while negative integers `n` are encoded as `2 * |n| - 1` (the odd numbers).
The encoding thus "zig-zags" between positive and negative numbers.
For example:

| Signed Original | Encoded As   |
|----------------:|:-------------|
| `0`             | 0            |
| `-1`            | 1            |
| `1`             | 2            |
| `-2`            | 3            |
| ...             | ...          |
| `0x7fffffff`    | `0xfffffffe` |
| `-0x80000000`   | `0xffffffff` |

In other words, each value `n` is encoded using

> (n << 1) ^ (n >> 31)

## Non-data message structure

This layout applies to all non-data messages

#### Structure

`START_BYTE | [ID] | [COUNT] | [DEPTH] | [TIME] | [END] | PATH`

#### Fields

Each message starts with a single byte `START_BYTE` that contains the following information:

| Field           | Start bit | Length | Content                                    |
|:----------------|:----------|:-------|:-------------------------------------------|
| `MESSAGE_TYPE`  | 0         | 3      | The [message type](#message-types)         |
| `HAS_ID`        | 3         | 1      | `1` if the header contains an `ID` field   |
| `HAS_COUNT`     | 4         | 1      | `1` if the header contains a `COUNT` field |
| `HAS_DEPTH`     | 5         | 1      | `1` if the header contains a `DEPTH` field |
| `HAS_TIME`      | 6         | 1      | `1` if the header contains a `TIME` field  |
| `HAS_END`       | 7         | 1      | `1` if the header contains an `END` field  |

Followed by the fields:

| Field   | Encoding           | Applicable message types                                        |
|:--------|:-------------------|:----------------------------------------------------------------|
| `ID`    | [Var-Int](#varint) | All non-data messages                                           |
| `COUNT` | [Var-Int](#varint) | [Request](#request-messages)                                    |
| `DEPTH` | [Var-Int](#varint) | All non-data messages                                           |
| `TIME`  | [DOUBLE](#value)   | [Request](#request-messages), [Delete](#data-deletion-messages) |
| `END`   | [DOUBLE](#value)   | [Request](#request-messages), [Delete](#data-deletion-messages) |
| `PATH`  | [PATH](#value)     | All non-data messages                                           |

## Request messages

Request messages are sent to SOIL components to request data.

#### Structure

[Non-data message structure](#non-data-message-structure)

#### Fields

| Field          | Description                                                    |
|:---------------|:---------------------------------------------------------------|
| `MESSAGE_TYPE` | `1`, see [message types](#message-types)                       |
| `ID`           | The unique id to identify the request                          |
| `COUNT`        | The maximum number of requested samples per node               |
| `DEPTH`        | The number of levels below the `path` for which to return data |
| `TIME`         | The start time of the requested interval (exclusive)           |
| `END`          | The end of the requested time interval (inclusive)             |
| `PATH`         | The requested top-level path for which data is requested       |

## Data deletion messages

These messages are used to request the deletion of data from supported nodes.

#### Structure

[Non-data message structure](#non-data-message-structure)

#### Fields

| Field          | Description                                                    |
|:---------------|:---------------------------------------------------------------|
| `MESSAGE_TYPE` | `2`, see [message types](#message-types)                       |
| `ID`           | Not applicable                                                 |
| `COUNT`        | Not applicable                                                 |
| `DEPTH`        | The number of levels below `path` to include in the deletion   |
| `TIME`         | The start time of the interval to delete (exclusive)           |
| `END`          | The end of the time interval to delete (inclusive)             |
| `PATH`         | The top-level path for which data is deleted                   |

## Subscription inclusion messages

These messages are used to subscribe to future updates.

#### Structure

[Non-data message structure](#non-data-message-structure)

#### Fields

| Field          | Description                                                      |
|:---------------|:-----------------------------------------------------------------|
| `MESSAGE_TYPE` | `3`, see [message types](#message-types)                         |
| `ID`           | Not applicable                                                   |
| `COUNT`        | Not applicable                                                   |
| `DEPTH`        | The number of levels below `path` to include in the subscription |
| `TIME`         | Not applicable                                                   |
| `END`          | Not applicable                                                   |
| `PATH`         | The top-level path to subscribe to                               |

## Subscription exclusion messages

These messages are used to unsubscribe from future updates.

#### Structure

[Non-data message structure](#non-data-message-structure)

#### Fields

| Field          | Description                                                        |
|:---------------|:-------------------------------------------------------------------|
| `MESSAGE_TYPE` | `4`, see [message types](#message-types)                           |
| `ID`           | Not applicable                                                     |
| `COUNT`        | Not applicable                                                     |
| `DEPTH`        | The number of levels below `path` to exclude from the subscription |
| `TIME`         | Not applicable                                                     |
| `END`          | Not applicable                                                     |
| `PATH`         | The top-level path to unsubscribe from                             |

## Subscription deletion messages

These messages are used to delete subscription rules.

#### Structure

[Non-data message structure](#non-data-message-structure)

#### Fields

| Field          | Description                                                         |
|:---------------|:--------------------------------------------------------------------|
| `MESSAGE_TYPE` | `5`, see [message types](#message-types)                            |
| `ID`           | Not applicable                                                      |
| `COUNT`        | Not applicable                                                      |
| `DEPTH`        | The number of levels below `path` for which to also clear the rules |
| `TIME`         | Not applicable                                                      |
| `END`          | Not applicable                                                      |
| `PATH`         | The top-level path for which to remove the subscription rules       |

## Examples

A simple data message with a non-timestamped value (`42`) at the root path:

| Header | Tree   | Sample | Value |
|:-------|:-------|:-------|:------|
| `00`   | `01`   | `01`   | `54`  |
