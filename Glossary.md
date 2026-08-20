
## Glossary

This document provides an overview of common terms used to describe `SOIL`.

#### Component

A single unit that acts according to the `SOIL` principles.
It may have `inputs` and/or `outputs`, and interacts with other components through messages.

#### State

A `state` is considered to be a single point of data over time, such as a temperature, a configuration setting, or any other piece of information that can change.
If a `state` is transmitted between components, then it is an [output](#output) for the sending component and an `input` for the receiver.
Each `state` is identified by a [path](#path), which denotes its location in the data tree structure.

#### Path

A path is a list of [path component](#path-component)s that serves as the identifier for a state.
This may be familiar from other systems, e.g. file system paths, except that `SOIL` purposefully treats a path as a list of components, rather than implicitly treating some separators (e.g. `/` or `.`) as special.
There are therefore no restrictions on the content of paths.

#### Path component

The building block of a `path`. In most `SOIL` systems a path component will be a string, so that a path could be `["some", "unique", "path"]`.
Other systems have similar concepts, except that a path component is allowed to be completely arbitrary within the limits of the defined type, e.g. a string path component can contain any characters, including common delimiters like `/` or `.`.

#### Output

An output is used by a [logic](#logic) to provide the results of its computations.
Each output is defined by a [path](#path) at which the data is published.

#### Input

An input is a [path](#path) that a [logic](#logic) observes to perform its computations.

#### Logic

A logic (block) is a SOIL component that transforms data from [inputs](#input) to [outputs](#output) according to some rules. There are many different types of logic, such as limit checkers, unit converters, path transformers, and many more.

#### Value

A value is a single piece of information located at a [path](#path). It can have one of several primitive types, such as `Int`, `String`, `Double`, `Data`, `Path`, or `Null`.

#### Timestamp

A timestamp is associated with a [value](#value), denoting the time of its creation. In most implementations, it is represented as a UTC timestamp, or as seconds since 1970.

#### Sample

A sample consists of a [value](#value) and a [timestamp](#timestamp). Samples are associated with [paths](#path) and can form a sample [tree](#tree).
Some components, like [storage](#storage) can hold many samples for a single path, to provide the history of a data point over time.

#### Tree

A tree consists of a number of nodes that can contain [samples](#sample), where each node can have child nodes identified by a [path component](#path-component). The list of components from the root node to the target node is called the [path](#path). Trees are the standard way of representing and transmitting data in `SOIL`.

#### Receiver

A receiver is a component that is capable of accepting `SOIL` data in the form of [trees](#tree).

#### Transmitter

A transmitter sends data (in the form of trees) to another `SOIL` component (a [receiver](#receiver)).

#### Provider

A provider is a `SOIL` component from which data can be requested.

#### Storage

A storage is a combination of a [receiver](#receiver) and a [provider](#provider), that also provides a `delete` operation. A storage can be a full database, or just something small that can store a limited amount of data.

#### Observable

An observable provides subscription services, by allowing a [receiver](#receiver) to subscribe to specific [paths](#path) for future updates.

#### Connection

A connection provides data transmission between two `SOIL` components, like a [receiver](#receiver) and a [transmitter](#transmitter). Connections can be realized over many different transport layers. There are message connections that transport `SOIL` messages, while others transport the [encoded](#encoding) messages, or even a binary stream containing [frames](#framing).

#### Message

Messages are used to transport `SOIL` data between components.
There are different types of messages, as documented in [Messages.md](Messages.md).

#### Encoding

An encoding is used to represent [messages](#message) as binary data.
There are a few standard encodings provided by SOIL, such as [JSON](Encodings/JSON.md) or [Protobuf](Encodings/protobuf/Protobuf.md), but custom encodings are also possible.

#### Framing

Framing is needed for [connections](#connection) where the underlying transport layer only provides a binary stream (without message boundaries), like a TCP socket. The framing is applied around [encoded](#encoding) messages to ensure that they can be detected in the stream.
