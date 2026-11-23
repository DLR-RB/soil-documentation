# SOIL (States/Outputs/Inputs/Logic)

`SOIL` is a general-purpose data distribution and management concept that aims to provide simplicity and flexibility.
It consists of:
- An abstract concept of how systems interact and exchange data
- A data distribution layer based on simple messages
- Concrete implementations and specifications for different transport layers and encodings
- Generic utilities and concepts for data management, storage, and processing
- Supporting code and documentation for fast system integration

## Core idea

`SOIL` is based on the idea that the data transmitted between/within systems can be represented using a simple tree structure containing primitive values.
The exchange of these structures is done using basic operations like write, read, delete, and subscribe.
A system can then be viewed as a collection of components that create and receive structured data.

Using this abstract view allows SOIL to be used as a simple data distribution framework between different systems through simple adapters.
The generic tree representation fits many use cases, and useful abstraction layers make it easy to reuse generic components.

On top of the simple data distribution SOIL provides concepts and implementations to perform logging, configuration, storage, subscriptions, data management, and even supports relational data models with multi-party synchronization.
Despite this myriad use cases, `SOIL`'s main aim is simplicity, by letting users choose which features to implement.

> The name `SOIL` comes from the idea that data organized in `states` (S), where `outputs` (O) are produced based on some `inputs` (I) according to some `logic` (L).

### Basic example

A simple environmental sensor may measure three properties: temperature, humidity, and pressure.
The sensor's data model may be represented as a small tree:


                     +----------+
                     |  Sensor  |
                     +----------+
                    /     |      \
                   /      |       \
    +-------------+  +----------+  +----------+
    | Temperature |  | Pressure |  | Humidity |
    +-------------+  +----------+  +----------+

A single temperature could therefore be transmitted by the sensor as:

```json
{
    "path": ["sensor", "temperature"],
    "value": 21.3
}
```

The message above is already a valid SOIL message in [JSON format](Docs/Encodings/JSON.md), allowing the sensor to send data into a SOIL system.
While the message contains only data for a single node in the data tree, it's also possible to send complex data structures, with different primitive types.

In addition to just sending data, it's also possible to request and delete data, and to subscribe to updates.
This allows more complex data flows. In the sensor example above, it would be possible to only explicitly request measurements from the sensor (polling),
or to subscribe to specific paths like the temperature.

### Philosophy

There are a few core principles guiding the development and use of `SOIL`:

#### Don't create another competing standard

There are already enough [competing standards](https://xkcd.com/927) out there.
`SOIL` is best thought of as an *idea* on how interactions between systems can be viewed at an abstract level.
It then establishes an abstract concept of data exchange, but still leaves room for variants to meet specific use cases.
Finally, it provides some implementations that can be applied for various purposes, or serve as examples for own implementations.
The intent is to create systems that can be *adapted* to work with existing standards through lightweight interfacing components.

#### Simple solutions for simple problems

When users want to do simple things (e.g. sending and receiving a single value), then the solution should also be simple.
No complex encodings, configuration, or setup. No unnecessary features that the user doesn't need for the task.

#### Composition over complexity

Many multi-purpose systems attempt to anticipate all use cases and supply solutions for everything in a single package.
This leads to very complex systems where most features are never used, and the users have to deal with complicated configuration, complex interfaces and extensive documentation.
Counter to this approach, `SOIL` encourages the perspective that all systems are composed of smaller components, which interact with each other.
By designing a system this way, it becomes easier to compose complex systems from simple components, tailored exactly to the current use case, but easily extensible in the future.

#### Dynamic by default, static if needed

It is common practice to "freeze" a system design before going into operations, so that the behaviour is more predictable.
This rigidity is often enforced in code or configuration, e.g. by only reading configuration at startup.
`SOIL` has a different approach: Everything is treated as dynamic by default, while static guarantees are enforced through other mechanisms, like access control on configuration changes.

## Further documentation

#### Important terminology

A brief description of the core terms can be found in the [glossary](Docs/Glossary.md).

#### Frequently asked questions

There is a living document to answer some common questions in the [FAQ](Docs/FAQ.md), which also explains some design decisions.

#### Subscriptions

An important mechanism of `SOIL` is the possibility to request updates from other components when new data is available.
This subscription system is further explained [here](Docs/Subscriptions.md).

#### Communication

SOIL defines abstract messages to exchange data between components.
These messages may be encoded in any form, and exchanged via any communication protocol.
Some components may send JSON messages over WebSockets, while others may communicate via custom protocols over proprietary RF links.

To support these scenarios, the messages must be:
- universal (no new messages for new use cases)
- independent (only implement required functionality)
- primitive (few fields, low complexity)

There are currently six different message types defined:
- Data: Send one or more values
- Request: Request current values or history
- Deletion: Request removal of data
- Subscriptions:
  - Inclusion: Add a tree to a subscription
  - Exclusion: Exclude a tree from a subscription
  - Deletion: Delete previous rules of a subscription

These messages are described in more detail in [Messages.md](Docs/Messages.md).

### Encodings

There are a few standardizes message encodings available for SOIL.
These are described in [Encodings](Docs/Encodings/Readme.md).

#### Access control

The main principles and recommendations for access control are compiled [here](Docs/Access%20control.md).

#### Security considerations

A short summary of security concepts and attacks related to SOIL can be found in [Security.md](Docs/Security.md).
