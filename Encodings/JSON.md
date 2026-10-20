# JSON Encoding

This document describes the JSON encoding format defined in the SOIL core libraries.

### Table of contents

- [Message types](#message-types)
- [Data messages](#data-messages)
- [Request messages](#request-messages)
- [Deletion requests](#data-deletion-messages)
- [Subscription inclusion](#subscription-inclusion-messages)
- [Subscription exclusion](#subscription-exclusion-messages)
- [Subscription deletion](#subscription-deletion-messages)
- [Message fields](#message-fields)
  - [Message type](#message-type)
  - [Path](#path)
  - [Value](#value)
  - [Value type](#value-type)
  - [Time](#time)
  - [Samples](#samples)
  - [Count](#count)
  - [Depth](#depth)
  - [End](#end)
  - [Id](#id)
  - [Children](#children)
- [Examples](#examples)

## Message types

There are currently six different messages in SOIL:

| Message Type                                              | Usage                                       |
|:----------------------------------------------------------|:--------------------------------------------|
| [Data message](#data-message)                             | Transmit a tree for a path                  |
| [Request message](#request-message)                       | Request a tree, including history           |
| [Deletion message](#data-deletion-message)                | Delete parts of a tree, including history   |
| [Subscription inclusion](#subscription-inclusion-message) | Request future updates for a path or tree   |
| [Subscription exclusion](#subscription-exclusion-message) | Prevent future updates for a path or tree   |
| [Subscription deletion](#subscription-deletion-message)   | Clear subscription rules for a path or tree |

### Data messages

Data messages are used to transmit data.
The following fields are supported:

| Field                      | Description                                                                 |
|:---------------------------|:----------------------------------------------------------------------------|
| [`message`](#message-type) | Omitted, or `data`                                                          |
| [`path`](#path)            | The top-level path of the transmitted data                                  |
| [`value`](#value)          | A value of the top-level `path`                                             |
| [`type`](#value-type)      | The data type of `value`                                                    |
| [`time`](#time)            | The timestamp of the `value`                                                |
| [`samples`](#samples)      | Samples associated with `path`                                              |
| [`count`](#count)          | Total number of grouped messages, if `id` is set                            |
| [`id`](#id)                | A unique id to identify the message, for responses or to group messages     |
| [`children`](#children)    | Additional data beneath the top level path. Used to construct complex trees |

### Request messages

Request messages are sent to SOIL components to request data.
The following fields are supported:

| Field                      | Description                                                    |
|:---------------------------|:---------------------------------------------------------------|
| [`message`](#message-type) | Must be `read`                                                 |
| [`path`](#path)            | The requested top-level path for which data is requested       |
| [`time`](#time)            | The start time of the requested interval (exclusive)           |
| [`count`](#count)          | The maximum number of requested samples per node               |
| [`depth`](#depth)          | The number of levels below the `path` for which to return data |
| [`end`](#end)              | The end of the requested time interval (inclusive)             |
| [`id`](#id)                | A unique id to identify the request, used in response message  |

### Data deletion messages

These messages are used to request the deletion of data from supported nodes.
The following fields are supported:

| Field                      | Description                                                  |
|:---------------------------|:-------------------------------------------------------------|
| [`message`](#message-type) | Must be `delete`                                             |
| [`path`](#path)            | The top-level path for which data is deleted                 |
| [`time`](#time)            | The start time of the interval to delete (exclusive)         |
| [`depth`](#depth)          | The number of levels below `path` to include in the deletion |
| [`end`](#end)              | The end of the time interval to delete (inclusive)           |

### Subscription inclusion messages

These messages are used to subscribe to future updates.
The following fields are supported:

| Field                      | Description                                                      |
|:---------------------------|:-----------------------------------------------------------------|
| [`message`](#message-type) | Must be `include`                                                |
| [`path`](#path)            | The top-level path to subscribe to                               |
| [`depth`](#depth)          | The number of levels below `path` to include in the subscription |

### Subscription exclusion messages

These messages are used to unsubscribe from future updates.
The following fields are supported:

| Field                      | Description                                                        |
|:---------------------------|:-------------------------------------------------------------------|
| [`message`](#message-type) | Must be `exclude`                                                  |
| [`path`](#path)            | The top-level path to unsubscribe from                             |
| [`depth`](#depth)          | The number of levels below `path` to exclude from the subscription |

### Subscription deletion messages

These messages are used to delete subscription rules.
The following fields are supported:

| Field                      | Description                                                         |
|:---------------------------|:--------------------------------------------------------------------|
| [`message`](#message-type) | Must be `clear`                                                     |
| [`path`](#path)            | The top-level path for which to remove the subscription rules       |
| [`depth`](#depth)          | The number of levels below `path` for which to also clear the rules |

## Message Fields

All six messages are realized using a single JSON message, with the following top-level fields:

| Field                      | Description                                                                       |
|:---------------------------|:----------------------------------------------------------------------------------|
| [`message`](#message-type) | Specifies the type of message                                                     |
| [`path`](#path)            | The top-level path                                                                |
| [`value`](#value)          | A value being transmitted                                                         |
| [`type`](#value-type)      | The data type of `value`                                                          |
| [`time`](#time)            | The timestamp of the `value`, or the start of a time interval                     |
| [`samples`](#samples)      | Samples associated with the `path`                                                |
| [`count`](#count)          | Total number of grouped messages, or maximum number of requested samples per node |
| [`depth`](#depth)          | Specifies the depth of a data request, deletion, or subscription                  |
| [`end`](#end)              | The end of a time interval for requests or deletions                              |
| [`id`](#id)                | A unique id to identify the message, for responses or to group messages           |
| [`children`](#children)    | Additional data beneath the top level path. Used to construct complex trees       |

### Message type

| Name          | `"message"` |
|:--------------|:------------|
| Type          | Text        |
| Optional      | Yes         |
| Default value | `"data"`    |

The `type` field identifies the type of the message, e.g. if the message represents a request for data, or transmits a tree.
The six different types of messages are represented by strings according to the following table:

| String    | Message Type                                               |
|:----------|:-----------------------------------------------------------|
| `data`    | [Data message](#data-message)                              |
| `read`    | [Request message](#request-message)                        |
| `delete`  | [Deletion message](#data-deletion-message)                 |
| `include` | [Subscription inclusion](#subscription-inclusion-message)  |
| `exclude` | [Subscription exclusion](#subscriptions-exclusion-message) |
| `clear`   | [Subscription deletion](#subscriptions-deletion-message)   |

The field is named `"message"` and expects a text value.

This field is optional. If the field is not contained in a message, then the message is treated as a data message (`data`).

Example:

```json
{
    "message": "data"
}
```

### Path

| Name          | `"path"` |
|:--------------|:---------|
| Type          | Text     |
| Optional      | Yes      |
| Default value | `[]`     |

The `path` field is used for all six message types:

| Type                                                      | Usage                                                                        |
|:----------------------------------------------------------|:-----------------------------------------------------------------------------|
| [Data message](#data-message)                             | The path to the root node of the tree in the message                         |
| [Request message](#request-message)                       | The path to the root node of the requested tree                              |
| [Deletion message](#data-deletion-message)                | The path to the root node of the data to delete                              |
| [Subscription inclusion](#subscription-inclusion-message) | The path to the root node of the tree to subscribe to                        |
| [Subscription exclusion](#subscription-exclusion-message) | The path to the root node of the tree to unsubscribe from                    |
| [Subscription deletion](#subscription-deletion-message)   | The path to the root node of the tree for which to delete subscription rules |

The field is named `"path"` and expects a JSON list of text values.
This field is optional. If the field is not present in a message, then an empty path is assumed.

Example:

```json
{
    "path": ["path", "of", "data"]
}
```

### Value

| Name          | `"value"`              |
|:--------------|:-----------------------|
| Type          | Text/Number/Null/Array |
| Optional      | Yes                    |
| Default value | N/A                    |

The `value` field is only used for [data messages](#data-message).
It contains a value associated with the path specified in the [`path`](#path) field.
The SOIL type of the contained value may be further specified with the [`type`](#value-type) field.
JSON types are mapped to SOIL types in the following way:

| SOIL Type | JSON Type               | [`type`](#value-type)      | Comment                                     |
|:----------|:------------------------|:---------------------------|:--------------------------------------------|
| Null      | `null`                  | omitted, or `null`         |                                             |
| Integer   | `Number`                | `"integer"`                |                                             |
| Double    | `Number`                | omitted, or `"double"`     |                                             |
| String    | `Text`                  | omitted, or `"string"`     |                                             |
| Data      | `Text`                  | `"data"`                   | String must contain base64-encoded data     |
| Path      | `Text` or `Array<Text>` | omitted, or `"path"`       | Text can be used for single component paths |
| List      | `Array<Object>`         | `"list"`                   | See [List](#list)                           |
| Map       | `Array<Object>`         | `"map"`                    | See [Map](#map)                             |

The value field can be used if only a single value/sample is transmitted for the path specified in the message.
It is permitted to transmit additional values using the [`samples`](#samples) field, or to only use the `samples` field.
An optional timestamp associated with the value is transmitted using the [`time`](#time) field.

Example:

```json
{
    "value": 123.4
}
```

#### List

Lists of values are encoded using an array of (type, value) pairs.
The type is the string associated with the stored value (see [Value Type](#value-type)),
while the value is the encoded value (see [Value](#value)).

Example:

```json
{
    "value": [
        "integer", 123,
        "path", ["a", "b"]
    ]
}
```

#### Map

Maps are encoded using an array of (key, type, value) triplets.
The key is a string, `type` is the [value type string](#value-type), and `value` is the encoded [Value](#value).

```json
{
    "value": [
        "abc", "integer", 123,
        "def", "list", ["string", "something"]
    ]
}
```

### Value Type

| Name          | `"type"` |
|:--------------|:---------|
| Type          | Text     |
| Optional      | Yes      |
| Default value | N/A      |

The `type` field is used only in connection with the `value` field, and identifies the type of the `value` field.
The `type` field is necessary to signal SOIL `Integer` and `Data` types, for all other types it is optional.
It can only appear without a `value` field, if `type` is set to the JSON `null` value.

| SOIL Type | JSON Type   |
|:----------|:------------|
| Null      | `null`      |
| Integer   | `"integer"` |
| Double    | `"double"`  |
| String    | `"string"`  |
| Data      | `"data"`    |
| Path      | `"path"`    |
| List      | `"list"`    |
| Map       | `"map"`     |

Example:

```json
{
    "type": "integer"
}
```

### Time

| Name          | `"time"`      |
|:--------------|:--------------|
| Type          | Number        |
| Optional      | Yes           |
| Default value | `null`        |

The `time` field contains an optional timestamp, encoded as a floating-point number of the seconds since 1.1.1970 (UTC).
It has different purposes depending on the message type:

| Type                                       | Usage                                                     | Default           |
|:-------------------------------------------|:----------------------------------------------------------|:------------------|
| [Data message](#data-message)              | The timestamp associated with the [`value`](#value) field | Current time      |
| [Request message](#request-message)        | The start time of the requested interval (exclusive)      | Beginning of time |
| [Deletion message](#data-deletion-message) | The start time of the interval to delete (exclusive)      | Beginning of time |

The `time` field is optional. If it is not present, then the default value in the table above is implied.

Example:

```json
{
    "time": 123456789.012
}
```

### Samples

| Name          | `"samples"`   |
|:--------------|:--------------|
| Type          | Array         |
| Optional      | Yes           |
| Default value | `[]`          |

The `samples` field only applies to [data messages](#data-message),
and can be used to transmit multiple samples (value + timestamp) for the path specified with the [`path`](#path) field.
The expected JSON structure for the field is an array of objects.
Each object represents a sample, with the following fields:

| Field   | Description                                                                           |
|:--------|:--------------------------------------------------------------------------------------|
| `value` | The value associated with the sample, see [value field](#value)                       |
| `type`  | The type of the value, see [value type](#value-type) (Note the different field name!) |
| `time`  | The timestamp associated with the value, see [time](#time)                            |

Samples use the same fields as the top-level object (`value`, `type`, `time`).
The following messages are equivalent:

```json
{
    "path": ["my", "value"],
    "value": 123,
    "type": "integer",
    "time": 123456789.012
}
```

```json
{
    "path": ["my", "value"],
    "samples": [
        {
            "value": 123,
            "type": "integer",
            "time": 123456789.012
        }
    ]
}
```

The former definition is available to make messages simpler in cases where only a single value is to be transmitted.

### Count

| Name          | `"count"` |
|:--------------|:----------|
| Type          | Number    |
| Optional      | Yes       |
| Default value | Null      |

The `count` field is applicable to the following message types:

| Type                                | Usage                                                                           |
|:------------------------------------|:--------------------------------------------------------------------------------|
| [Data message](#data-message)       | The number of messages of the current request (only if [`id`](#id) is not null) |
| [Request message](#request-message) | The maximum number of samples to return for each requested tree node            |

The field is optional. If it is not present, then a `null` value is assumed, indicating:
- For data messages: The number of grouped messages is unknown or not provided by the sender
- For request messages: All samples in the specified time frame should be returned

### Depth

| Name          | `"depth"` |
|:--------------|:----------|
| Type          | Number    |
| Optional      | Yes       |
| Default value | Null      |

The `depth` field is applicable to the following messages:

| Type                                                      | Usage                                                                                    |
|:----------------------------------------------------------|:-----------------------------------------------------------------------------------------|
| [Request message](#request-message)                       | The number of levels below the specified path to provide in the response                 |
| [Deletion message](#data-deletion-message)                | The number of levels below the specified path to delete                                  |
| [Subscription inclusion](#subscription-inclusion-message) | The number of levels below the specified path to subscribe to                            |
| [Subscription exclusion](#subscription-exclusion-message) | The number of levels below the specified path to unsubscribe from                        |
| [Subscription deletion](#subscription-deletion-message)   | The number of levels below the specified path for which to delete the subscription rules |

The field is optional. If it is not specified, then it implies an infinite depth.
Negative numbers are treated as if the value is zero.

### End

| Name          | `"end"`   |
|:--------------|:----------|
| Type          | Number    |
| Optional      | Yes       |
| Default value | Null      |

The `end` field contains an optional timestamp, encoded as a floating-point number of the seconds since 1.1.1970 (UTC).
It signifies the end of a time interval in the following messages:

| Type                                       | Usage                                                | Default     |
|:-------------------------------------------|:-----------------------------------------------------|:------------|
| [Request message](#request-message)        | The end time of the requested interval (inclusive)   | End of time |
| [Deletion message](#data-deletion-message) | The start time of the interval to delete (inclusive) | End of time |

The `time` field is optional. If it is not present, then the default value in the table above is implied.

Example:

```json
{
    "end": 123456789.012
}
```

### Id

| Name          | `"id"` |
|:--------------|:----------|
| Type          | Number    |
| Optional      | Yes       |
| Default value | Null      |

The `id` field can be used with the following messages:

| Type                                | Usage                                                                                                                        |
|:------------------------------------|:-----------------------------------------------------------------------------------------------------------------------------|
| [Data message](#data-message)       | The id of the request to which this message responds, or an id of a single data transmission spread across multiple messages |
| [Request message](#request-message) | The id of the request, to associate the responses with                                                                       |

The field is optional. If it is not present, then no id is used.
Responses to unidentified requests are then sent without a response, or a random id.
Data messages without an id can be further processed without waiting for other messages.

### Children

| Name          | `"children"` |
|:--------------|:-------------|
| Type          | Object       |
| Optional      | Yes          |
| Default value | `{}`         |

The `children` field is only applicable to [Data messages](#data-message) and can be used to provide additional samples beneath the top-level [path](#path).
This allows the transmission of complex tree structures.
The content of the field is a JSON object, where each field name corresponds to the name of a child, and the value is another JSON object:

```json
{
    "children": {
        "child1": {

        },
        "child2": {

        }
    }
}
```

Each object in this nested structure represents a node in a tree, and can have the following fields:

| Field      | Description                                                                           |
|:-----------|:--------------------------------------------------------------------------------------|
| `value`    | A value associated with this node, see [value field](#value)                          |
| `type`     | The type of the value, see [value type](#value-type) (Note the different field name!) |
| `time`     | The timestamp associated with the value, see [time](#time)                            |
| `samples`  | An array of samples for this node, see [samples](#samples)                            |
| `children` | Additional child nodes of this node                                                   |

This recursive definition allows the transmission of complex tree structures, with multiple levels of nesting.

## Examples

Consider the following information:
```
rover.position.x = 12.3
rover.position.y = 23.4
rover.position.x.target = 15.0
rover.velocity.x = 34.5
rover.velocity.y = 45.6
```

Encoded in a single message:

```json
{
    "path": ["rover"],
    "children": {
        "position": {
            "children": {
                "x": {
                    "value": 12.3,
                    "children": {
                        "target": {
                            "value": 15.0
                        },
                    }
                },
                "y": {
                    "value": 23.4
                }
            }
        },
        "velocity": {
            "children": {
                "x": {
                    "value": 34.5
                },
                "y": {
                    "value": 45.6
                }
            }
        }
    }
}
```

It's also possible to encode multiple samples for each node, and include timestamps as needed.