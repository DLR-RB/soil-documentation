# Protobuf encoding

This document describes the SOIL message encoding using [Google Protocol Buffers](https://protobuf.dev).
The corresponding message definitions can be found in [soil.proto](soil.proto).

### Table of contents

- [Message types](#message-types)
- [Data messages](#data-messages)
- [Request messages](#request-messages)
- [Deletion requests](#data-deletion-messages)
- [Subscription inclusion](#subscription-inclusion-messages)
- [Subscription exclusion](#subscription-exclusion-messages)
- [Subscription deletion](#subscription-deletion-messages)

## Message types

There are currently six different messages in SOIL, defined in a protobuf enum:

```proto
enum PbMessageType {
    // Transmit a tree for a path (default if absent)
    DATA = 0;

    // Request a tree, including history
    REQUEST = 1;

    // Delete parts of a tree, including history
    DELETE = 2;

    // Request future updates for a path or tree
    SUBSCRIPTION_INCLUDE = 3;

    // Stop future updates for a path or tree
    SUBSCRIPTION_EXCLUDE = 4;

    // Clear subscription rules for a path or tree
    SUBSCRIPTION_CLEAR = 5;
}
```

## Message structure

All SOIL messages are encoded using the same protobuf message:

```proto
message PbMessage {
    MessageType message = 1;
    repeated string path = 2;
    optional double start_time = 3;
    optional double end_time = 4;
    optional uint32 count = 5;
    optional uint32 depth = 6;
    optional uint64 id = 7;
    optional Tree data = 8;
}
```

> Most fields are declared optional: Not setting the field indicates a `null` value,
> which is an important feature for the different message types.
> Decoders must check if the field was explicitly set, to distinguish "no value" from the default.
> Encoders must not set those fields to default values.

## Data messages

Data messages use the following fields of a [generic message](#message-structure):

| Field   | Content                                                                 |
|:--------|:------------------------------------------------------------------------|
| `type`  | Set to `DATA`, see [message types](#message-types)                      |
| `path`  | The top-level path of the transmitted `data`                            |
| `count` | Total number of grouped messages, if `id` is set                        |
| `id`    | A unique id to identify the message, for responses or to group messages |
| `data`  | The transmitted data, encoded as a [tree](#tree)                        |

Note that the fields `id`, and `count` can be `null`, in which case the respective field is not set.

### Tree

Trees are used in [data messages](#data-messages), and are defined recursively:

```proto
message PbTree {
    // The samples at the node
    repeated PbSample samples = 8;

    // The children of that node
    map<string, PbTree> children = 9;
}
```

Each tree can contain a list of [samples](#sample), and a number of child trees, which are identified by their string component.

### Sample

Samples are attached to [trees](#tree), and contain a value and an optional timestamp:

```proto
message PbSample {
    optional double time = 1;

    oneof value {
        sint64 integer = 2;
        double double = 3;
        string string = 4;
        bytes data = 5;
        PbPath path = 6;
    }
}
```

The `time` is encoded as a [timestamp](#timestamp) (UTC seconds since 1.1.1970).

> A path is encoded as `PbPath`, which contains a single field with a repeated string,
> since protobuf `oneof` can't directly contain repeated fields.

### Value

As specified [above](#sample), values are encoded using a Protobuf `oneof`,
which allows only one field to be set.
The SOIL `null` value is encoded by setting none of the `value` fields.

### Timestamp

A timestamp defined as a `double` value of the UTC seconds since January 1st, 1970.

## Request messages

Request messages are sent to SOIL components to request data.
The following fields of a [generic message](#message-structure) are used:

| Field        | Description                                                    |
|:-------------|:---------------------------------------------------------------|
| `type`       | Set to `REQUEST`, see [message types](#message-types)          |
| `id`         | The unique id to identify the request                          |
| `count`      | The maximum number of requested samples per node               |
| `depth`      | The number of levels below the `path` for which to return data |
| `start_time` | The start time of the requested interval (exclusive)           |
| `end_time`   | The end of the requested time interval (inclusive)             |
| `path`       | The requested top-level path for which data is requested       |

Note that the fields `id`, `count`, `depth`, `start_time` and `end_time` can be `null`,
in which case the respective field is not set.

## Data deletion messages

These messages are used to request the deletion of data from supported nodes.
The following fields of a [generic message](#message-structure) are used:

| Field        | Description                                                    |
|:-------------|:---------------------------------------------------------------|
| `type`       | Set to `DELETE`, see [message types](#message-types)           |
| `depth`      | The number of levels below `path` to include in the deletion   |
| `start_time` | The start time of the interval to delete (exclusive)           |
| `end_time`   | The end of the time interval to delete (inclusive)             |
| `path`       | The top-level path for which data is deleted                   |

Note that `depth`, `start_time` and `end_time` can be `null`, in which case the respective field is not set.

## Subscription inclusion messages

These messages are used to subscribe to future updates.
The following fields of a [generic message](#message-structure) are used:

| Field   | Description                                                        |
|:--------|:-------------------------------------------------------------------|
| `type`  | Set to `SUBSCRIPTION_INCLUDE`, see [message types](#message-types) |
| `depth` | The number of levels below `path` to include in the subscription   |
| `path`  | The top-level path to subscribe to                                 |

Note that `depth` can be `null`, in which case the field is not set.

## Subscription exclusion messages

These messages are used to unsubscribe from future updates.
The following fields of a [generic message](#message-structure) are used:

| Field   | Description                                                        |
|:--------|:-------------------------------------------------------------------|
| `type`  | Set to `SUBSCRIPTION_EXCLUDE`, see [message types](#message-types) |
| `depth` | The number of levels below `path` to exclude from the subscription |
| `path`  | The top-level path to unsubscribe from                             |

Note that `depth` can be `null`, in which case the field is not set.

## Subscription deletion messages

These messages are used to delete subscription rules.
The following fields of a [generic message](#message-structure) are used:

| Field   | Description                                                         |
|:--------|:--------------------------------------------------------------------|
| `type`  | Set to `SUBSCRIPTION_CLEAR`, see [message types](#message-types)    |
| `depth` | The number of levels below `path` for which to also clear the rules |
| `path`  | The top-level path for which to remove the subscription rules       |

Note that `depth` can be `null`, in which case the field is not set.
