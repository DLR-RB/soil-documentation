# Binary Encoding

This document describes the binary encoding format implemented in the SOIL core libraries.

### Table of contents

- [Conventions](#conventions)
- [Message types](#message-types)
- [Message structure](#message-structure)
  - [Header](#header)
  - [Header fields](#header-fields)
- [Data messages](#data-messages)
  - [Tree](#tree)
  - [Child](#child)
  - [Sample](#sample)
- [Request messages](#request-messages)
- [Deletion requests](#data-deletion-messages)
- [Subscription inclusion](#subscription-inclusion-messages)
- [Subscription exclusion](#subscription-exclusion-messages)
- [Subscription deletion](#subscription-deletion-messages)
- [Primitives](#primitives)
  - [Value](#value)
  - [Value type](#value-type)
  - [Timestamp](#timestamp)
  - [VarInt](#varint)
  - [ZigZag VarInt](#zigzag-varint)
- [Decoding rules](#decoding-rules)
- [Examples](#examples)
- [Value encoding](#value-encoding)

## Conventions

Bits within a byte are numbered starting from the most significant bit (MSB), i.e. bit `0` is the MSB, and bit `7` is the least significant bit.
All bytes are written as hexadecimal numbers, e.g. `F6`.

## Message types

There are currently six different messages in SOIL:

| Type                                                       | Raw Value | Usage                                       |
|:-----------------------------------------------------------|:----------|:--------------------------------------------|
| [Data message](#data-messages)                             |       `0` | Transmit a tree for a path                  |
| [Request message](#request-messages)                       |       `1` | Request a tree, including history           |
| [Deletion message](#data-deletion-messages)                |       `2` | Delete parts of a tree, including history   |
| [Subscription inclusion](#subscription-inclusion-messages) |       `3` | Request future updates for a path or tree   |
| [Subscription exclusion](#subscription-exclusion-messages) |       `4` | Prevent future updates for a path or tree   |
| [Subscription deletion](#subscription-deletion-messages)   |       `5` | Clear subscription rules for a path or tree |

The raw values `6` and `7` are reserved, and messages with these types are rejected.

## Message structure

All messages share the same structure:

`HEADER | [ID] | [COUNT] | [DEPTH] | [TIME] | [END] | PATH | [TREE]`

The fields in brackets are optional. Their presence is indicated by flags in the header.
The `TREE` is only present in [data messages](#data-messages).

### Header

Each message starts with a single byte `HEADER` that contains the following information:

| Field          | Start bit | Length | Content                                      |
|:---------------|:----------|:-------|:---------------------------------------------|
| `MESSAGE_TYPE` | 0         | 3      | The [message type](#message-types)           |
| `HAS_ID`       | 3         | 1      | `1` if the message contains an `ID` field    |
| `HAS_COUNT`    | 4         | 1      | `1` if the message contains a `COUNT` field  |
| `HAS_DEPTH`    | 5         | 1      | `1` if the message contains a `DEPTH` field  |
| `HAS_TIME`     | 6         | 1      | `1` if the message contains a `TIME` field   |
| `HAS_END`      | 7         | 1      | `1` if the message contains an `END` field   |

The header byte is therefore `MESSAGE_TYPE << 5`, combined with the flags `10` (`ID`), `08` (`COUNT`), `04` (`DEPTH`), `02` (`TIME`) and `01` (`END`).

> For a data message without optional fields, the header byte is just `00`.

### Header fields

The header is followed by the fields in this order:

| Field   | Encoding                  | Currently used by                                                                       |
|:--------|:--------------------------|:----------------------------------------------------------------------------------------|
| `ID`    | [VarInt](#varint)         | [Data](#data-messages), [Request](#request-messages)                                    |
| `COUNT` | [VarInt](#varint)         | [Data](#data-messages), [Request](#request-messages)                                    |
| `DEPTH` | [VarInt](#varint)         | All messages except [data messages](#data-messages)                                     |
| `TIME`  | [Timestamp](#timestamp)   | [Data](#data-messages), [Request](#request-messages), [Delete](#data-deletion-messages) |
| `END`   | [Timestamp](#timestamp)   | [Request](#request-messages), [Delete](#data-deletion-messages)                         |
| `PATH`  | [Path](#value)            | All messages                                                                            |

The `PATH` is always present. An empty path is encoded as the single byte `00`.

The `ID` is a 64-bit signed integer, encoded as a [VarInt](#varint) of its two's complement bit pattern.
Negative ids are therefore always encoded with 9 bytes.

`COUNT` and `DEPTH` are unsigned. Senders encode negative values as `0`.
Values that are too large for the receiver are clamped to the largest supported value.

All header fields can be used with every message type, even if they currently have no meaning for that type, e.g. an `ID` in a deletion message.
Encoders write every field that is set, and decoders read every field that is present, regardless of the message type.
Since the presence of each field is indicated by the header, such messages can always be decoded without ambiguity,
and encoders and decoders don't need to change if the fields are used by more message types in the future.
Components ignore fields that they don't use.

The `TREE` is not a header field, and is only encoded in data messages.

## Data messages

Data messages are used to transmit data.

#### Structure

[Message structure](#message-structure), followed by a `TREE`.

#### Fields

| Field          | Description                                                                                      |
|:---------------|:-------------------------------------------------------------------------------------------------|
| `MESSAGE_TYPE` | `0`, see [message types](#message-types)                                                         |
| `ID`           | The id of the request to which this message responds, or an id of a data transmission spread across multiple messages |
| `COUNT`        | The total number of messages of the current request or group                                     |
| `DEPTH`        | Currently unused, see [header fields](#header-fields)                                            |
| `TIME`         | The timestamp of the message, e.g. when it was created, or when a request was processed          |
| `END`          | Currently unused, see [header fields](#header-fields)                                            |
| `PATH`         | The path of the root node of the `TREE`                                                          |
| `TREE`         | The data being transmitted, see [tree](#tree)                                                    |

Unlike the [JSON encoding](JSON.md#time), the `TIME` field never applies to a sample.
Timestamps of samples are encoded within each [sample](#sample).

### Tree

#### Structure

`TREE_HEADER | [EXTENDED_CHILD_COUNT] | [EXTENDED_SAMPLE_COUNT] | [SAMPLES] | [CHILD_NODES]`

#### Fields

The `TREE_HEADER` is a single byte:

| Field          | Start bit | Length | Content                                                             |
|:---------------|:----------|:-------|:--------------------------------------------------------------------|
| `CHILD_COUNT`  | 0         | 4      | The number of child nodes in this node, or `0b1111` if 15 or more   |
| `SAMPLE_COUNT` | 4         | 4      | The number of samples for this node, or `0b1111` if 15 or more      |

It is followed by these fields:

| Field                   | Encoding              | Content                                                                    |
|:------------------------|:----------------------|:---------------------------------------------------------------------------|
| `EXTENDED_CHILD_COUNT`  | [VarInt](#varint)     | The total number of child nodes, only present if `CHILD_COUNT = 0b1111`    |
| `EXTENDED_SAMPLE_COUNT` | [VarInt](#varint)     | The total number of samples, only present if `SAMPLE_COUNT = 0b1111`       |
| `SAMPLES`               | [Sample](#sample)     | The samples of this node, one after another                                |
| `CHILD_NODES`           | [Child](#child)       | The children of this node, one after another                               |

The extended counts are only needed for tree nodes that have 15 or more samples or children.
They will thus rarely be needed, so that the header of most nodes is a single byte.

> An empty tree is encoded as `00`. A tree with only one sample simplifies to `01 | SAMPLE`.

### Child

#### Structure

`NAME | TREE`

#### Fields

| Field  | Encoding            | Content                                  |
|:-------|:--------------------|:-----------------------------------------|
| `NAME` | [String](#value)    | The name of the child                    |
| `TREE` | [Tree](#tree)       | The data of the child                    |

Senders write the children sorted by name, so that the encoding is deterministic.
Receivers accept children in any order.
If a name appears multiple times in a node, then the last child is used.

### Sample

#### Structure

`SAMPLE_HEADER | [TIMESTAMP] | VALUE`

#### Fields

The `SAMPLE_HEADER` is a single byte:

| Field           | Start bit | Length | Content                                                  |
|:----------------|:----------|:-------|:---------------------------------------------------------|
| `HAS_TIMESTAMP` | 0         | 1      | `1` if the sample contains a `TIMESTAMP` field           |
| `VALUE_TYPE`    | 1         | 7      | The type of the value, see [value type](#value-type)     |

It is followed by these fields:

| Field       | Encoding                  | Content                                                          |
|:------------|:--------------------------|:-----------------------------------------------------------------|
| `TIMESTAMP` | [Timestamp](#timestamp)   | The timestamp of the sample, only present if `HAS_TIMESTAMP = 1` |
| `VALUE`     | [Value](#value)           | The content of the value (empty for `null`)                      |

A sample without a timestamp is decoded with a `null` timestamp, which is interpreted as the current time.
Decoders do not replace a missing timestamp with the current time.

> For a sample without a timestamp, the first byte simply denotes the type, e.g. `01` for an integer.
> With a timestamp, the type is combined with `80`, e.g. `81` for an integer.

## Request messages

Request messages are sent to SOIL components to request data.

#### Structure

[Message structure](#message-structure)

#### Fields

| Field          | Description                                                                              |
|:---------------|:-----------------------------------------------------------------------------------------|
| `MESSAGE_TYPE` | `1`, see [message types](#message-types)                                                 |
| `ID`           | The unique id to identify the request, used in response messages                         |
| `COUNT`        | The maximum number of requested samples per node. Missing: All samples                   |
| `DEPTH`        | The number of levels below the `PATH` for which to return data. Missing: Infinite depth  |
| `TIME`         | The start time of the requested interval (exclusive). Missing: Beginning of time         |
| `END`          | The end of the requested time interval (inclusive). Missing: End of time                 |
| `PATH`         | The requested top-level path for which data is requested                                 |

## Data deletion messages

These messages are used to request the deletion of data from supported nodes.

#### Structure

[Message structure](#message-structure)

#### Fields

| Field          | Description                                                                            |
|:---------------|:---------------------------------------------------------------------------------------|
| `MESSAGE_TYPE` | `2`, see [message types](#message-types)                                               |
| `ID`           | Currently unused, see [header fields](#header-fields)                                  |
| `COUNT`        | Currently unused, see [header fields](#header-fields)                                  |
| `DEPTH`        | The number of levels below `PATH` to include in the deletion. Missing: Infinite depth  |
| `TIME`         | The start time of the interval to delete (exclusive). Missing: Beginning of time       |
| `END`          | The end of the time interval to delete (inclusive). Missing: End of time               |
| `PATH`         | The top-level path for which data is deleted                                           |

## Subscription inclusion messages

These messages are used to subscribe to future updates.

#### Structure

[Message structure](#message-structure)

#### Fields

| Field          | Description                                                                                |
|:---------------|:-------------------------------------------------------------------------------------------|
| `MESSAGE_TYPE` | `3`, see [message types](#message-types)                                                   |
| `ID`           | Currently unused, see [header fields](#header-fields)                                      |
| `COUNT`        | Currently unused, see [header fields](#header-fields)                                      |
| `DEPTH`        | The number of levels below `PATH` to include in the subscription. Missing: Infinite depth  |
| `TIME`         | Currently unused, see [header fields](#header-fields)                                      |
| `END`          | Currently unused, see [header fields](#header-fields)                                      |
| `PATH`         | The top-level path to subscribe to                                                         |

## Subscription exclusion messages

These messages are used to unsubscribe from future updates.

#### Structure

[Message structure](#message-structure)

#### Fields

| Field          | Description                                                                                  |
|:---------------|:---------------------------------------------------------------------------------------------|
| `MESSAGE_TYPE` | `4`, see [message types](#message-types)                                                     |
| `ID`           | Currently unused, see [header fields](#header-fields)                                        |
| `COUNT`        | Currently unused, see [header fields](#header-fields)                                        |
| `DEPTH`        | The number of levels below `PATH` to exclude from the subscription. Missing: Infinite depth  |
| `TIME`         | Currently unused, see [header fields](#header-fields)                                        |
| `END`          | Currently unused, see [header fields](#header-fields)                                        |
| `PATH`         | The top-level path to unsubscribe from                                                       |

## Subscription deletion messages

These messages are used to delete subscription rules.

#### Structure

[Message structure](#message-structure)

#### Fields

| Field          | Description                                                                                   |
|:---------------|:----------------------------------------------------------------------------------------------|
| `MESSAGE_TYPE` | `5`, see [message types](#message-types)                                                      |
| `ID`           | Currently unused, see [header fields](#header-fields)                                         |
| `COUNT`        | Currently unused, see [header fields](#header-fields)                                         |
| `DEPTH`        | The number of levels below `PATH` for which to also clear the rules. Missing: Infinite depth  |
| `TIME`         | Currently unused, see [header fields](#header-fields)                                         |
| `END`          | Currently unused, see [header fields](#header-fields)                                         |
| `PATH`         | The top-level path for which to remove the subscription rules                                 |

## Primitives

### Value

Values are used in [samples](#sample), where the type is stored in the `SAMPLE_HEADER`, followed by the content of the value.
Elements of lists and maps are tagged values, i.e. the [value type](#value-type) as a single byte, followed by the content.

| SOIL Type | [`VALUE_TYPE`](#value-type) | Content                                                                          |
|:----------|:----------------------------|:---------------------------------------------------------------------------------|
| Null      | `0`                         | No data                                                                          |
| Integer   | `1`                         | 64-bit signed integer as [ZigZag VarInt](#zigzag-varint)                         |
| Double    | `2`                         | 64-bit IEEE 754 double, big endian                                               |
| String    | `3`                         | Byte count as [VarInt](#varint), followed by the UTF-8 bytes                     |
| Data      | `4`                         | Byte count as [VarInt](#varint), followed by the bytes                           |
| Path      | `5`                         | Component count as [VarInt](#varint), followed by the components as `String`     |
| List      | `6`                         | Element count as [VarInt](#varint), followed by the elements as `TYPE + CONTENT` |
| Map       | `7`                         | Entry count as [VarInt](#varint), followed by the entries as `KEY + TYPE + CONTENT`, where `KEY` is a `String` |

Doubles can be any IEEE value, including NaN and infinity.
Senders write map entries sorted by key, so that the encoding is deterministic.
Receivers accept entries in any order. If a key appears multiple times, then the last entry is used.

Examples:

| Value                 | Content (excluding the type)      |
|:----------------------|:----------------------------------|
| Null                  | *No data*                         |
| `123` (Integer)       | `F6 01`                           |
| `3.14` (Double)       | `40 09 1E B8 51 EB 85 1F`         |
| `"abc"` (String)      | `03 61 62 63`                     |
| `0xABBA` (Data)       | `02 AB BA`                        |
| `"a", "b"` (Path)     | `02 01 61 01 62`                  |
| `["a", 123]` (List)   | `02 03 01 61 01 F6 01`            |
| `{ "a" : 123 }` (Map) | `01 01 61 01 F6 01`               |

### Value type

The type of a value is identified by a number:

| SOIL Type | `VALUE_TYPE` |
|:----------|:-------------|
| Null      | `0`          |
| Integer   | `1`          |
| Double    | `2`          |
| String    | `3`          |
| Data      | `4`          |
| Path      | `5`          |
| List      | `6`          |
| Map       | `7`          |

Other numbers are reserved, and values with unknown types are rejected.

### Timestamp

A timestamp is the number of seconds since January 1st, 1970 (UTC), encoded as a 64-bit IEEE 754 double in big endian byte order.
Non-finite timestamps (NaN and infinity) are not allowed.

### VarInt

VarInts are variable-length encodings of unsigned 64-bit integers, using between 1 and 9 bytes.
They are similar to the varints of [Google Protocol Buffers](https://protobuf.dev/programming-guides/encoding/#varints):
The value is split into groups of 7 bits, starting with the least significant group.
Each group is stored in the lower 7 bits of a byte, and the most significant bit (MSB) of the byte indicates if another byte follows.

Values between 0 and 127 are encoded using a single byte:

| Value | Encoding     |
|:------|:-------------|
| `0`   | `0b00000000` |
| `1`   | `0b00000001` |
| `2`   | `0b00000010` |
| ...   | ...          |
| `126` | `0b01111110` |
| `127` | `0b01111111` |

Unlike Protocol Buffers, a VarInt has at most 9 bytes:
After 8 bytes with 7 bits each (56 bits), the 9th byte has no continuation bit, and contains the 8 most significant bits of the value.
The encoding is therefore identical to Protocol Buffers for values below 2^56.

To decode a VarInt:

```
value = 0
for shift in 0, 7, 14, 21, 28, 35, 42, 49:
    byte = nextByte()
    value = value | ((byte & 0b01111111) << shift)
    if (byte & 0b10000000) == 0:
        return value
byte = nextByte()
return value | (byte << 56)
```

To encode a VarInt:

```
repeat 8 times:
    if value < 0b10000000:
        writeByte(value)
        return
    writeByte((value & 0b01111111) | 0b10000000)
    value = value >> 7
writeByte(value)
```

For example, the number `150` is encoded as `96 01`, or `10010110 00000001`:

```
10010110 00000001        // Original inputs.
 0010110  0000001        // Drop continuation bits.
 0000001  0010110        // Reverse the order of the groups (most significant group first).
   00000010010110        // Concatenate.
 128 + 16 + 4 + 2 = 150  // Interpret as an unsigned 64-bit integer.
```

More examples:

| Value          | Encoding                     |
|:---------------|:-----------------------------|
| `128`          | `80 01`                      |
| `300`          | `AC 02`                      |
| `2^56 - 1`     | `FF FF FF FF FF FF FF 7F`    |
| `2^56`         | `80 80 80 80 80 80 80 80 01` |
| `2^64 - 1`     | `FF FF FF FF FF FF FF FF FF` |

### ZigZag VarInt

Signed integer values are encoded as [VarInt](#varint), except that non-negative integers `p` are encoded as `2 * p` (the even numbers), while negative integers `n` are encoded as `2 * |n| - 1` (the odd numbers).
The encoding thus "zig-zags" between positive and negative numbers, so that numbers with a small magnitude are encoded with few bytes.
For example:

| Signed Original         | Encoded As              | Bytes                        |
|------------------------:|:------------------------|:-----------------------------|
| `0`                     | `0`                     | `00`                         |
| `-1`                    | `1`                     | `01`                         |
| `1`                     | `2`                     | `02`                         |
| `-2`                    | `3`                     | `03`                         |
| `123`                   | `246`                   | `F6 01`                      |
| ...                     | ...                     | ...                          |
| `0x7fffffffffffffff`    | `0xfffffffffffffffe`    | `FE FF FF FF FF FF FF FF FF` |
| `-0x8000000000000000`   | `0xffffffffffffffff`    | `FF FF FF FF FF FF FF FF FF` |

In other words, each 64-bit value `n` is encoded using

> (n << 1) ^ (n >> 63)

where `>>` is an arithmetic shift. It is decoded using `(z >> 1) ^ -(z & 1)`, where `>>` is a logical shift.

## Decoding rules

The following rules apply to all messages:

- **Unknown types:** Messages with an unknown [message type](#message-types), and samples or values with an unknown [value type](#value-type), are rejected.
- **Truncated data:** Messages that end before all fields are read are rejected.
- **Trailing data:** Messages with additional bytes after the end of the message are rejected.
- **Counts:** The counts of strings, data, paths, lists, maps, samples and children can't be larger than the number of remaining bytes, since each element requires at least one byte. Larger counts are rejected.
- **Nesting:** Receivers may limit the nesting depth of trees, lists and maps, to prevent excessive resource usage. The SOIL core libraries reject messages with more than 512 levels.
- **Strings:** Senders must write valid UTF-8. Receivers replace invalid byte sequences with the replacement character (`U+FFFD`).

## Examples

A data message with a non-timestamped integer value (`42`) at the root path:

| Header | Path | Tree | Sample | Value |
|:-------|:-----|:-----|:-------|:------|
| `00`   | `00` | `01` | `01`   | `54`  |

A data message with the double `1.5` for the path `["a"]`, with the timestamp `1700000000.125`:

| Header | Path       | Tree | Sample | Timestamp                 | Value                     |
|:-------|:-----------|:-----|:-------|:--------------------------|:--------------------------|
| `00`   | `01 01 61` | `01` | `82`   | `41 D9 54 FC 40 08 00 00` | `3F F8 00 00 00 00 00 00` |

A data message with two children, `a = 1` and `b = null`, at the root path:

| Header | Path | Tree | Name `a` | Tree | Sample | Value | Name `b` | Tree | Sample |
|:-------|:-----|:-----|:---------|:-----|:-------|:------|:---------|:-----|:-------|
| `00`   | `00` | `20` | `01 61`  | `01` | `01`   | `02`  | `01 62`  | `01` | `00`   |

A request message with the id `7` for the path `["sensor"]` with depth `2`:

| Header | Id   | Depth | Path                         |
|:-------|:-----|:------|:-----------------------------|
| `34`   | `07` | `02`  | `01 06 73 65 6E 73 6F 72`    |

A subscription to all data (empty path, infinite depth):

| Header | Path |
|:-------|:-----|
| `60`   | `00` |

## Value encoding

Single values can also be encoded independently of messages, e.g. to store them in files or databases.
The same representation as for values within messages is used.
Two variants exist:

- **Tagged values** contain the [value type](#value-type) as a single byte, followed by the [content](#value) of the value.
  This is the same representation as a single element of a list.
  For example, the integer `123` is encoded as `01 F6 01`.
- **Value content** only contains the [content](#value) of a value, if the type is known from another source (e.g. a database column).
  For example, the integer `123` is encoded as `F6 01`, and `null` as no data.

In both cases, no other data may follow the value.
