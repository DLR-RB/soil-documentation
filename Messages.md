# Messages

SOIL defines abstract messages to exchange data between components.
These messages may be encoded in any form, and exchanged via any communication protocol.
Some components may send JSON messages over WebSockets, while others may communicate via custom protocols over proprietary RF links.

To support these scenarios, the messages must be:
- universal (no new messages for new use cases)
- independent (only implement required functionality)
- primitive (few fields, low complexity)

## Message types

There are currently three different message types defined:

| Type                                | Usage                                |
|:------------------------------------|:-------------------------------------|
| [Value message](#value-message)     | Transmit values for paths            |
| [Data request](#request-message)    | Request values, create subscriptions |
| [History request](#history-request) | Request historic values for paths    |

### Value message

The simplest and most common message type is used to transfer data, in the form of values associated with paths.

A value message has the following fields:

| Field     | Type                      | Description                                                                                 |
|:----------|:--------------------------|:--------------------------------------------------------------------------------------------|
| `path`    | `Array<String>`           | The path to which the value belongs.                                                        |
| `value`   | `Optional<Value>`         | The value being transmitted. Can be omitted if `samples` is set.                            |
| `time`    | `Optional<Date>`          | The timestamp of the `value`. Current time is assumed if omitted.                           |
| `samples` | `Optional<Array<Sample>>` | The list of values being transmitted. Can be omitted if `value` is provided. Each sample consists of a `value` and a `time` (both required). |
| `request` | `Optional<Int>`           | The request id to which this message is a response                                          |
| `count`   | `Optional<Int>`           | An optional number stating the total number of responses sent for the request               |

Value messages are used to transmit new or historic values of paths.
Single `value`s are usually transmitted based on subscriptions, or shared unrequested by a data producer.

The simplest message to provide data (e.g. by sensors) only contains the `path` and a `value`.
It is therefore very easy to adapt simple devices and inject data into the system.

Note that `value` has the unspecified type `Value`. The abstract SOIL concept does not specify which types of values the system handles.
Some use cases may call for highly specific types, while others may just use basic types like integers, doubles, and strings.
It is recommended to keep the types as primitive as possible, to keep the complexity of the implementations low.
There is also often no need to handle type sizes (like `Int32` or `UInt8`) within SOIL, since those are mostly restrictions for transmissions and storage, which don't apply to SOIL.

#### Responses

Value messages are used as responses to requests (described below).
Each request can produce an arbitrary number of responses (including zero).
If a request includes a `request` field, then this value should be contained in the corresponding responses, if supported by the component.
Responses to data or history requests should also include a `count` of all messages generated for the request, so that the receiving component can determine the completion of a request.
If no `count` is supplied, then the component receiving the updates has to decide when the request is completed, e.g. after some timeout.
If `count` is supplied, then the component will consider the request completed once `count` messages have been received.

#### Acknowledgements

It possible to use value messages as acknowledgements, should this functionality be absolutely required (the transport layer is assumed to deliver all messages).
If a component receives a value message containing a `request` field that is not a response to a request made by itself, then it should respond with a value message containing the `path` of the received message, with the `time` field set to the current time.
The `value` and `samples` fields should be left empty.

### Data request message

The second message type is used to request data from another component.
These requests can concern a single value, but can also target a larger section of the path tree, and are also used to create and manage subscriptions to paths.

 > Note: Not all components may support request messages. If requests are not supported, then the messages will simply be ignored.

A request message has the following fields:

| Field       | Type                 | Description                                                                                          |
|:------------|:---------------------|:-----------------------------------------------------------------------------------------------------|
| `path`      | `Array<String>`      | The path to be requested.                                                                            |
| `depth`     | `Optional<Int>`      | The number of levels below the path to include. All values of the sub-tree are requested if omitted. |
| `subscribe` | `Optional<Bool>`     | Omitted to request a single value. If `true`, then a subscription is established. If `false`, then the sub-tree is excluded from the subscriptions. |
| `time`      | `Optional<Date>`     | The point in time of the requested path. Current time is assumed if omitted.                         |
| `request`   | `Optional<Int>`      | The request id to identify the response(s)                                                           |

If `request` is present in the message, then components are expected to include the same value in all responses related to this request, except for future updates due to subscriptions.

Additionally, the `count` field may be set by the components for each response, to allow the requesting component to track that all responses have been received.
If `count` is supplied, then the component will consider the request completed once `count` messages have been received.
If no `count` is supplied, then the component receiving the updates has to decide when the request is completed, e.g. after some timeout.

#### Single value requests

The `subscribe` property is omitted to get the current value of a path (or all values of a sub-tree).
If `depth` is set to `0`, then the component should only return the value for the exact `path` provided.
If `depth` is set to `nil`, then the full sub-tree below (and including) `path` is requested.
If `depth` is set to a positive value, then the specified number of levels below `path` are also requested.
If the path `a, b` is requested with a depth of `1`, then `a, b, c` should also be returned, but not `a, b, c, d`.

If a `time` is specified, then the newest values at or before the given date should be returned.

If the `request` field is present, then components should respond with at least one message.
If no data is available for the request, then the requested path should be sent without any `value` or `samples`.
The response may include a timestamp to indicate when the response was created.

The `count` property can be set in responses to value requests if `request` is set, to indicate the total number of messages being sent.
If `count` is supplied, then the component will consider the request completed once `count` messages have been received.
If no `count` is supplied, then the component receiving the updates has to decide when the request is completed, e.g. after some timeout.

#### Subscription requests

If the `subscribe` property is set to `true`, then the component is requested to send all future changes to those paths using [value messages](#value-message), and with all changes to those paths in the future.
The component is not expected to respond with the current values.
If the `request` field is set in the request, then the component may respond with an empty [value message](#value-message) containing the received path and request id.

The `depth` property has the same meaning as for [single value requests](#single-value-requests).

#### Unsubscribing

To cancel a subscription, send a request message with `subscribe` set to `false`, using the same `path` and `depth` as the original subscription.
If `request` is set, then the component should respond with an empty [value message](#value-message) containing the path and request id.

#### Exclusion requests

Subscribing to a sub-tree may cause unnecessary updates, so it is also possible to exclude sub-trees within subscriptions from being published.
This is done by sending a request message with `subscribe` set to `false`, indicating to the receiver that the specified sub-tree should be excluded.
This exclusion mechanism can be used together with subscriptions to create fine-grained update strategies.
For more information, see the [subscriptions document](subscriptions.md).

The component also responds with a value message if the `request` field is set, consistent with [subscription requests](#subscription-requests).

### History request

A history request is conceptually similar to a single value request, but instead of requesting the current value, it instructs the receiver to send historic values for the path within an interval.

 > Note: Not all components may support history requests. If not supported, then the messages will simply be ignored.

A history request message has the following fields:

| Field     | Type             | Description                                                                          |
|:----------|:-----------------|:-------------------------------------------------------------------------------------|
| `path`    | `Array<String>`  | The path for which history is requested                                              |
| `start`   | `Optional<Date>` | The start date of the interval (exclusive). Beginning of time is assumed if omitted. |
| `end`     | `Optional<Date>` | The end date of the interval (inclusive). End of time is assumed if omitted          |
| `limit`   | `Optional<Int>`  | The maximum number of values to transmit                                             |
| `request` | `Optional<Int>`  | The request id to identify the response(s)                                           |

Components that support history requests should respond with [value messages](#value-message) containing the requested samples.
The samples may arrive through individual messages (via `value`), or as batches using `samples`.

#### Range intervals and limiting

Depending how (and if) `start` and `end` are specified, the component is expect to respond with different samples.
In general, values are returned from `start` to `end`, until the optional `limit` is reached.
If `start` is not specified, then beginning of time is assumed, and if `end` is omitted, then end of time is assumed.
This holds true even if `start` is later than `end`, meaning that the `limit` newest samples will be returned.

Note that `start` is treated as exclusive, while `end` is inclusive.
This has been choosen so that clients can continue their queries from the last available sample.

If the `request` field is set, then the component receiving the request should include it in all messages being sent as a response,
including a `count` stating the number of messages being sent.

#### Response order

The components are not required to respond with the samples in any particular order (this may also not be guaranteed by the transport layer),
but it is recommended to send them from `start` to `end`.

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