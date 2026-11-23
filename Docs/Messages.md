# Messages

SOIL defines abstract messages to exchange data between components.
These messages may be encoded in different forms, and exchanged via different communication protocols.
Some components may send JSON messages over WebSockets, while others may communicate via custom protocols over proprietary RF links.

To support these scenarios, the messages are designed to be:
- universal (no new messages for new use cases)
- independent (only implement required functionality)
- primitive (few fields, low complexity)

The message structures in this document are only described conceptually.
SOIL doesn't define the exact arrangement, identification or encoding of the messages.
Check specific implementations for details on these aspects.

### Notation

In this document, the following data types are used:
- `Optional<T>`: A type that either contains a value of type `T`, or no value (e.g. `null`, `nil`, `None`)
- `Array<T>`: An ordered list of elements of type `T`. May contain an arbitrary number of elements, including zero.
- `Map<K,V>`: A mapping of keys of type `K` to values of type `V`. Each key only exists once. May contain an arbitrary number of key-value pairs, and may be empty.
- `Date`: A representation of a point in time. Includes date and time information.
- `UInt`: An unsigned integer number. May be zero, but not negative. The allowed value range depends on the implementation.

## Message types

There are currently these different message types defined:

| Type                                       | Usage                               |
|:-------------------------------------------|:------------------------------------|
| [Data message](#data-message)              | Transmit samples for paths          |
| [Request message](#request-message)        | Request samples, including history  |
| [Deletion message](#data-deletion-message) | Delete samples, including history   |
| [Subscription inclusion](#subscriptions)   | Include a path in a subscription    |
| [Subscription exclusion](#subscriptions)   | Exclude a path from a subscription  |
| [Subscription deletion](#subscriptions)    | Delete rules of a subscription      |

## Data message

All possible fields in a data message can be found in the following table.

A value message has the following fields:

| Field     | Type                       | Description                                                                   |
|:----------|:---------------------------|:------------------------------------------------------------------------------|
| `path`    | `Optional<Array<String>>`  | The path to which the value belongs. Defaults to an empty array               |
| `data`    | `Optional<SampleTree>`     | The data being transmitted. Defaults to an empty tree                         |
| `request` | `Optional<UInt>`           | The request id to which this message is a response                            |
| `count`   | `Optional<UInt>`           | An optional number stating the total number of responses sent for the request |

The `data` field consists of a sample tree, as defined in the [Data Model](Model.md).

## Data request message

The second message type is used to request data from another component.
These requests can concern a single value, but can also target a larger section of the path tree, and are also used to query historic data.
Each request can produce an arbitrary number of responses (including zero).

 > Note: Not all components may support request messages. If requests are not supported, then the messages will simply be ignored.

A request message has the following fields:

| Field     | Type             | Description                                                                                                       |
|:----------|:-----------------|:------------------------------------------------------------------------------------------------------------------|
| `path`    | `Array<String>`  | The path to be requested.                                                                                         |
| `start`   | `Optional<Date>` | The start date of the interval (exclusive). Beginning of time is assumed if omitted.                              |
| `end`     | `Optional<Date>` | The end date of the interval (inclusive). End of time is assumed if omitted                                       |
| `limit`   | `Optional<UInt>` | The maximum number of samples to include for each node in the tree. All matching samples are returned if omitted. |
| `depth`   | `Optional<UInt>` | The number of levels below the path to include. All values of the sub-tree are requested if omitted.              |
| `request` | `Optional<UInt>` | The request id to identify the response(s)                                                                        |

### Single value requests

If `depth == 0`, then the component should only return the value for the exact `path` provided.
If `depth` is set to `nil`, then the full sub-tree below (and including) `path` is requested.
If `depth` is set to a positive value, then the specified number of levels below `path` are also requested.
If the path `a, b` is requested with a depth of `1`, then `a, b, c` should also be returned, but not `a, b, c, d`.

If a `time` is specified, then the newest values at or before the given date should be returned.

### Range intervals and limiting

Depending how (and if) `start` and `end` are specified, the component is expect to respond with different samples.
In general, values are returned from `start` to `end`, until the optional `limit` is reached.
If `start` is not specified, then beginning of time is assumed, and if `end` is omitted, then end of time is assumed.
This holds true even if `start` is later than `end`, meaning that the `limit` newest samples will be returned.
If an `end` but no `start` is specified, then `limit` will apply to the *newest* samples.

Note that `start` is treated as exclusive, while `end` is inclusive.
This has been choosen so that clients can continue their queries from the last available sample.

If the `request` field is set, then the component receiving the request should include it in all messages being sent as a response,
including a `count` stating the number of messages being sent.

### Response order

The components are not strictly required to respond with the samples in any particular order, and are also free to split the responses into an arbitrary number of messages.
It is recommended to use a fixed batch size, and to sort the `samples` within the messages in the expected order (mostly from `start` to `end`).

### Request identification

If `request` is present in the message, then components are expected to include the same value in all responses related to this request.
Additionally, the `count` field may be set by the components for each response, to allow the requesting component to track that all responses have been received.
If `count` is supplied, then the component will consider the request completed once `count` messages have been received.
If no `count` is supplied, then the component receiving the updates has to decide when the request is completed, e.g. after some timeout.
It is valid (but not advised) for `count` to increase with subsequent messages in the same request, e.g. if a component can't fully determine the number of messages in a request before starting the response.

If the `request` field is present, then components should respond with at least one message.
If no data is available for the request, then the requested path should be sent without any data.
The response may include a timestamp to indicate when the response was created.

## Data deletion message

The deletion of data is requested in almost the same way as a data request.

| Field     | Type             | Description                                                                                          |
|:----------|:-----------------|:-----------------------------------------------------------------------------------------------------|
| `path`    | `Array<String>`  | The path to be deleted                                                                               |
| `start`   | `Optional<Date>` | The start date of the interval (exclusive). Beginning of time is assumed if omitted.                 |
| `end`     | `Optional<Date>` | The end date of the interval (inclusive). End of time is assumed if omitted                          |
| `depth`   | `Optional<UInt>` | The number of levels below the path to delete. All values of the sub-tree are deleted if omitted.    |
| `request` | `Optional<UInt>` | The request id to identify the response(s)                                                           |

## Subscriptions

Subscriptions are an optional SOIL feature to request continous updates from components when new values at specified paths are available.
Establishing and managing subscriptions is done with a separate set of messages, while the actual data updates use the [data messages](#data-message) defined above.

> Subscriptions are an optional feature not supported by all components. Subscription messages are ignored if not supported.

Subscriptions define areas of the data tree that the component is interested in.
This is done by including or excluding sub-trees with an optional depth, to construct the desired subscription pattern.
There are three operations:
- Include a sub-tree in the subscription
- Exclude a sub-tree from the subscription
- Delete the entire subscription

Including and excluding paths will create the subscription, if it doesn't exist yet.

Subscriptions are configured per component, meaning that each component can establish individual subscriptions.
Each subscription is identified by an integer id, which can be used to update or delete the subscription later.

For a more detailed description of the features, see [subscriptions](Subscriptions.md).

### Subscription rules

Both messages for inclusion and exclusion have the following fields:

| Field          | Type             | Description                                                                                                 |
|:---------------|:-----------------|:------------------------------------------------------------------------------------------------------------|
| `path`         | `Array<String>`  | The path to include or exclude                                                                              |
| `depth`        | `Optional<UInt>` | The number of levels below the path to target. The full tree below the path is included/excluded if omitted |

### Deleting subscription rules

A subscription deletion request can be used to remove all inclusion and exclusion rules of a subscription in a subtree.

| Field          | Type             | Description                                     |
|:---------------|:-----------------|:------------------------------------------------|
| `path`         | `Array<String>`  | The path for which to delete subscription rules |
| `depth`        | `Optional<UInt>` | The number of levels below the path to target. `nil` indicates that all rules at and below the path should be deleted |

To completely remove all subscriptions, issue a request with an empty path (the root path) and no depth (`nil`).

## Additional Notes

### Message implementations

Systems using SOIL are free to implement these messages however they see fit.
Key-value based formats like [JSON](Encodings/JSON.md) and [Protobuf](Encodings/protobuf/Protobuf.md) are available, as are location-based formats (like a [binary format](Encodings/Binary.md)).
It is also left to the implementations to distinguish the message types from one another.
There a various ways to achieve this, and SOIL provides no restrictions here.

### Transport layer requirements

The protocols and implementations to exchange messages can be freely chosen, but SOIL makes some assumptions that are relevant for developers implementing communication layers for SOIL messages.

#### Arbitrary message size

Due to its generic nature, a single SOIL message may grow arbitrarily large.
The transport layer should allow the transmission of messages up to a reasonable upper limit that depends on the concrete mission requirements.
Alternatively, it may split the binary data of messages, and recombine it on the receiver side.
The core implementations of SOIL provide components to split large data messages into smaller parts, grouped using a request id.
This feature may not be supported by all messages.

#### Data integrity & completeness

It is assumed that the transport layer provides adequate guarantees that the messages are delivered without modifications, e.g. that no data is modified, deleted, or added.
This is usually done through integrity checks, error correction, and retransmission.

#### Lossless delivery

It is assumed that messages between components are delivered without loss, i.e. all messages sent are also delivered.

#### Message order

There are no assumptions about the message order. Messages may be delivered in any order to the components receiving them.
The SOIL mechanisms are designed to handle all data exchange as atomic operations, or provides functions to track out-of-order messages, like counters.
