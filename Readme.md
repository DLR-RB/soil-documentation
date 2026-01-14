# SOIL (States/Outputs/Inputs/Logic)

`SOIL` is a general-purpose data management, transmission, and monitoring/control concept, which aims to provide ease-of-use and flexibility while creating complex systems that exchange, store, and display data from various sources.
All data in the system is envisioned as a tree structure, with components publishing and subscribing to paths in this tree to synchronize their data.
This generic approach can be applied to many different areas, including spacecraft operations, sensor networks, server monitoring, file system backups, configuration management, and distributed logging.
It's even possible to build relational data models, complete with multi-party syncronization.
Despite this myriad use cases, `SOIL`'s main aim is simplicity.

Some highlights of what `SOIL` can provide:
- Exchange of arbitrary data structures through simple messages
- Archiving with no mission-specific setup
- Treatment of logging and configuration as data
- Freedom to use different transport layers and message encoding
- Subscriptions and requests to create complex data flows
- Transmission and management of historic data
- Dynamic configuration with persistence, history, and reporting
- Dynamic system composition and extension

Perhaps equally important is what it does *not* do:
- It doesn't force you to use a specific platform or programming language
- It doesn't force you to use a specific transport layer
- It doesn't force you to use a specific message encoding
- It doesn't tell you how to structure your data
- It doesn't tell you how to structure your data flows
- It doesn't tell you how to structure your system

`SOIL` provides:
- A unified view of many different use cases with the same abstract properties
- Instructions to convert and adapt systems to work in a generic way
- Generic components to send, receive, and archive data
- Implementations of the core concepts for multiple programming languages and platforms

### Core concepts

The framework is based on a few main axioms:
- Systems can be structured as (small) components, which interact with each other through data exchange.
- The data of each component can be broken down into primitive data types with some structure.

Within `SOIL`, every part of the system is viewed as a component with data organized in `states` (S), which produces some `outputs` (O) based on some `inputs` (I) according to some internal `logic` (L) (hence the name `SOIL`).
These inputs and outputs are the interfaces to other components.
Each input/output represents a single `state`, which changes its `value` over time, e.g. the temperature reading of a sensor.
The output of one component can be the input of one or more other components (e.g. for telemetry), and it's also possible for multiple components to write to a single input (e.g. for commanding or configuration).
States are uniquely identified by `paths`, which consist of a list of strings, e.g. `["device", "sensor", "temperature"]`.
This creates a tree structure, where `["sensor", "temperature"]` and `["sensor", "voltage"]` are both sub-elements of `["sensors"]`. This concept should feel familiar, e.g. from file systems or MQTT.

Each data point (`state`) in the system is intended as a primitive type (think integer or string), but can be arbitrarily complex.
The components exchange updates for `state`s via simple messages, which also facilitate operations like subscriptions or the request of historical data.
With these simple primities

### Philosophy

There are a few core principles guiding the development and use of `SOIL`:

#### Don't create another competing standard

There are already enough [competing standards](https://xkcd.com/927) out there.
`SOIL` is best thought of as an *idea* on how interactions between systems can be viewed at an abstract level.
It then establishes an abstract concept of data exchange, but still leaves room for variants to meet specific use cases.
Finally, it provides some implementations that can be applied for various purposes, or serve as examples for own implementations.
The intent is to create systems that can be *adapted* to work with existing standards through lightweight interfacing components.
It should also be easily possible to connect different variants based on `SOIL`.

#### Simple solutions for simple problems

When users want to do simple things (e.g. sending and receiving a single value), then the solution should also be simple.
No complex encodings, configuration, or setup. No unnecessary features that the user doesn't need for the task.
As an example, here is how a simple sensor value is transmitted in `SOIL`:

```json
{
    "path": ["sensor", "temperature"],
    "value": 21.3
}
```

There is no need to understand subscriptions, history requests or other concepts, since they are not relevant here.

#### Composition over complexity

Many multi-purpose systems attempt to anticipate all use cases and supply solutions for everything in a single package.
This leads to very complex systems where most features are never used, and the users have to deal with complicated configuration, complex interfaces and extensive documentation.
Counter to this approach, `SOIL` encourages the perspective that all systems are composed of smaller components, which interact with each other.
By designing a system this way, it becomes easier to compose complex systems from simple components, tailored exactly to the current use case, but easily extensible in the future

#### Dynamic by default, static if needed

It is common practice to "freeze" a system design before going into operations, so that the behaviour is more predictable.
This rigidity is often enforced in code or configuration, e.g. by only reading configuration at startup.
`SOIL` has a different approach: Everything is treated as dynamic by default, while static guarantees are enforced through other mechanisms, like access control on configuration changes.

## Further documentation

#### Important terminology

A brief description of the core terms can be found in the [glossary](Glossary.md).

#### Frequently asked questions

There is a living document to answer some common questions in the [FAQ](FAQ.md), which also explains some design decisions.

#### Subscriptions

An important mechanism of `SOIL` is the possibility to request updates from other components when new data is available.
This subscription system is further explained [here](Subscriptions.md).

#### Communication

SOIL defines abstract messages to exchange data between components.
These messages may be encoded in any form, and exchanged via any communication protocol.
Some components may send JSON messages over WebSockets, while others may communicate via custom protocols over proprietary RF links.

To support these scenarios, the messages must be:
- universal (no new messages for new use cases)
- independent (only implement required functionality)
- primitive (few fields, low complexity)

There are currently three different message types defined:
- Push: Send a value
- Request: Request values, create subscriptions
- History: Request historic values in an interval

These messages are described in more detail in [Messages.md](Messages.md).

