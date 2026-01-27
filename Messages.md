# Messages

SOIL defines abstract messages to exchange data between components.
These messages may be encoded in any form, and exchanged via any communication protocol.
Some components may send JSON messages over WebSockets, while others may communicate via custom protocols over proprietary RF links.

To support these scenarios, the messages must be:
- universal (no new messages for new use cases)
- independent (only implement required functionality)
- primitive (few fields, low complexity)

### Notes

The message structures in this document are only described conceptually.
SOIL doesn't define the exact arrangement, identification or encoding of the messages.
Check specific implementations for details on these aspects.

There are currently these different message types defined:

| Type                                      | Usage                               |
|:------------------------------------------|:------------------------------------|
| [Data message](#data-message)             | Transmit samples for paths          |
| [Request message](#request-message)       | Request samples, including history  |
| [Deletion message](#data-deletion-message) | Delete samples, including history |
| [Subscription inclusion](#subscriptions)   | Include a path in a subscription   |
| [Subscription exclusion](#subscriptions)   | Exclude a path from a subscription |
| [Subscription deletion](#subscriptions)    | Deletrules of a subscription       |

## Core functionality

At the most basic level, a SOIL component only does two things: Transmit and receive data.
Consequently, there are only two messages required to perform these functions.

### Sending data

At an abstract level, the data sent to a receiver contains:
- The `path` of the transmitted data in the data tree
- A list of `samples` with the timestamped values for the path

A simple message would therefore look like this:

```json
{
    "path" : ["sensor", "temperature"],
    "samples" : [
        {
            "value" : 14.3,
            "timestamp" : 1769422796.19203
        }
    ]
}
```

Data messages are used to transmit new or historic values of paths.
Single `value`s are usually transmitted based on subscriptions, or shared unrequested by a data producer.



### Single value messages

In most systems, the majority of transmissions will only contain a single sample,
which merits the addition of more fields to simplify messages in those cases:
- A `value` field at the top level for the singular value
- An optional `timestamp` field at the top level for the timestamp of the value

A single-value message can therefore be simply:

```json
{
    "path" : ["sensor", "temperature"],
    "value" : 14.3,
    "timestamp" : 1769422796.19203
}
```

If the sender of a value wants the receiver to treat the value as belonging to the current time,
then the `timestamp` can be omitted (e.g. when a sensor has no suitable clock).
In this case the message is simply:

```json
{
    "path" : ["sensor", "temperature"],
    "value" : 14.3
}
```

### Responses

In the above case the message was being sent without a request, e.g. because a sensor measured a new value.
Data messages may also be sent as a response to some request for data, e.g. querying a database.
In this case, the requestor may want to know which request the data messages belong to.
We therefore add two more optional fields to data messages:
- A `request` id that matches an id in the request that this message responds to.
- A `count` that indicates how many total messages will be sent in response to the request.

If a request includes a `request` field, then this value should be contained in the corresponding responses, if supported by the component.
Responses to data or history requests should also include a `count` of all messages generated for the request, so that the receiving component can determine the completion of a request.
If no `count` is supplied, then the component receiving the updates has to decide when the request is completed, e.g. after some timeout.
If `count` is supplied, then the component will consider the request completed once `count` messages have been received.

### Data message

All possible fields in a data message can be found in the following table.

A value message has the following fields:

| Field     | Type                      | Description                                                                                 |
|:----------|:--------------------------|:--------------------------------------------------------------------------------------------|
| `path`    | `Array<String>`           | The path to which the value belongs.                                                        |
| `value`   | `Optional<Value>`         | The value being transmitted. Can be omitted if `samples` is set.                            |
| `time`    | `Optional<Date>`          | The timestamp of the `value`. Current time is assumed if omitted.                           |
| `samples` | `Optional<Array<Sample>>` | The list of values being transmitted. Can be omitted if `value` is provided. Each sample consists of a `value` and a `time` (both required). |
| `request` | `Optional<Int>`           | The request id to which this message is a response                                          |
| `count`   | `Optional<Int>`           | An optional number stating the total number of responses sent for the request               |

Note that `value` has the unspecified type `Value`. The abstract SOIL concept does not specify which types of values the system handles.
Some use cases may call for highly specific types, while others may just use basic types like integers, doubles, and strings.
It is recommended to keep the types as primitive as possible, to keep the complexity of the implementations low.
There is also often no need to handle type sizes (like `Int32` or `UInt8`) within SOIL, since those are mostly restrictions for transmissions and storage, which don't apply to SOIL.

#### Acknowledgements

It possible to use value messages as acknowledgements, should this functionality be absolutely required (the transport layer is assumed to deliver all messages).
If a component receives a value message containing a `request` field that is not a response to a request made by itself, then it should respond with a value message containing the `path` of the received message, with the `time` field set to the current time.
The `value` and `samples` fields should be left empty.

### Data request message

The second message type is used to request data from another component.
These requests can concern a single value, but can also target a larger section of the path tree, and are also used to query historic data.
Each request can produce an arbitrary number of responses (including zero).

 > Note: Not all components may support request messages. If requests are not supported, then the messages will simply be ignored.

A request message has the following fields:

| Field     | Type             | Description                                                                                          |
|:----------|:-----------------|:-----------------------------------------------------------------------------------------------------|
| `path`    | `Array<String>`  | The path to be requested.                                                                            |
| `start`   | `Optional<Date>` | The start date of the interval (exclusive). Beginning of time is assumed if omitted.                 |
| `end`     | `Optional<Date>` | The end date of the interval (inclusive). End of time is assumed if omitted                          |
| `limit`   | `Optional<Int>`  | The maximum number of samples to include for each node in the tree. All matching samples are returned if omitted. |
| `depth`   | `Optional<Int>`  | The number of levels below the path to include. All values of the sub-tree are requested if omitted. |
| `request` | `Optional<Int>`  | The request id to identify the response(s)                                                           |

#### Single value requests

If `depth <= 0`, then the component should only return the value for the exact `path` provided.
If `depth` is set to `nil`, then the full sub-tree below (and including) `path` is requested.
If `depth` is set to a positive value, then the specified number of levels below `path` are also requested.
If the path `a, b` is requested with a depth of `1`, then `a, b, c` should also be returned, but not `a, b, c, d`.

If a `time` is specified, then the newest values at or before the given date should be returned.

#### Range intervals and limiting

Depending how (and if) `start` and `end` are specified, the component is expect to respond with different samples.
In general, values are returned from `start` to `end`, until the optional `limit` is reached.
If `start` is not specified, then beginning of time is assumed, and if `end` is omitted, then end of time is assumed.
This holds true even if `start` is later than `end`, meaning that the `limit` newest samples will be returned.
If an `end` but no `start` is specified, then `limit` will apply to the *newest* samples.

Note that `start` is treated as exclusive, while `end` is inclusive.
This has been choosen so that clients can continue their queries from the last available sample.

If the `request` field is set, then the component receiving the request should include it in all messages being sent as a response,
including a `count` stating the number of messages being sent.

#### Response order

The components are not strictly required to respond with the samples in any particular order, and are also free to split the responses into an arbitrary number of messages.
It is recommended to use a fixed batch size, and to sort the `samples` within the messages in the expected order (mostly from `start` to `end`).

#### Request identification

If `request` is present in the message, then components are expected to include the same value in all responses related to this request.
Additionally, the `count` field may be set by the components for each response, to allow the requesting component to track that all responses have been received.
If `count` is supplied, then the component will consider the request completed once `count` messages have been received.
If no `count` is supplied, then the component receiving the updates has to decide when the request is completed, e.g. after some timeout.

If the `request` field is present, then components should respond with at least one message.
If no data is available for the request, then the requested path should be sent without any `value` or `samples`.
The response may include a timestamp to indicate when the response was created.

The `count` property can be set in responses to value requests if `request` is set, to indicate the total number of messages being sent.
If `count` is supplied, then the component will consider the request completed once `count` messages have been received.
If no `count` is supplied, then the component receiving the updates has to decide when the request is completed, e.g. after some timeout.

### Data deletion message

The deletion of data is requested in almost the same way as a data request.

| Field     | Type             | Description                                                                                          |
|:----------|:-----------------|:-----------------------------------------------------------------------------------------------------|
| `path`    | `Array<String>`  | The path to be deleted  .                                                                            |
| `start`   | `Optional<Date>` | The start date of the interval (exclusive). Beginning of time is assumed if omitted.                 |
| `end`     | `Optional<Date>` | The end date of the interval (inclusive). End of time is assumed if omitted                          |
| `depth`   | `Optional<Int>`  | The number of levels below the path to delete. All values of the sub-tree are deleted if omitted.    |
| `request` | `Optional<Int>`  | The request id to identify the response(s)                                                           |

### Subscriptions

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

#### Subscription rules

Both messages for inclusion and exclusion have the following fields:

| Field          | Type            | Description                                    |
|:---------------|:----------------|:-----------------------------------------------|
| `subscription` | `Int`           | The id of the subscription to create or modify |
| `path`         | `Array<String>` | The path to include or exclude                 |
| `depth`        | `Optional<Int>` | The number of levels below the path to target. The full tree below the path is included/excluded if omitted |

#### Deleting subscriptions

A subscription deletion request can be used to remove all inclusion and exclusion rules of a subscription in a subtree.

| Field          | Type            | Description                                     |
|:---------------|:----------------|:------------------------------------------------|
| `subscription` | `Int`           | The id of the subscription to modify            |
| `path`         | `Array<String>` | The path for which to delete subscription rules |
| `depth`        | `Optional<Int>` | The number of levels below the path to target. `nil` indicates that all rules at and below the path should be deleted |

To completely remove all subscriptions, issue a request with an empty path (the root path) and no depth (`nil`).

## Message implementations

Systems using SOIL are free to implement these messages however they see fit.
Key-value based formats like JSON are possible, as are location-based formats.
It is also left to the implementations to distinguish the message types from one another.
There a various ways to achieve this, and SOIL provides no restrictions or suggestions here.

### Transport layer requirements

The protocols and implementations to exchange messages can be freely chosen, but SOIL makes some assumptions that are relevant for developers implementing communication layers for SOIL messages.

#### Arbitrary message size

Due to its generic nature, a single SOIL message may grow arbitrarily big.
The transport layer should allow the transmission of messages up to a reasonable upper limit that depends on the concrete mission requirements.

#### Data integrity & completeness

It is assumed that the transport layer provides adequate guarantees that the messages are delivered without modifications, e.g. that no data is modified, deleted, or added.
This is usually done through integrity checks, error correction, and retransmission.

#### Lossless delivery

It is assumed that messages between components are delivered without loss, i.e. all messages sent are also delivered.

#### Message order

There are no assumptions about the message order. Messages may be delivered in any order to the components receiving them.
The SOIL mechanisms are designed to handle all data exchange as atomic operations, or provides functions to track out-of-order messages, like counters.