# FAQ

Some commonly asked questions and design decisions are collected here.

#### How are components identified?

SOIL doesn't specify any format or structure for component identifiers.
The identifiers for components are likely mission-specific, or depend on the transport layer (IP Addresses, domain names, node ids...).
The only requirement for component ids is that they can be used to determine where messages are supposed to be sent, e.g. for subscriptions.

#### Why are paths defined as lists, and not as a single string like in file systems?

The common practice of using separators like `/` or `.` to split paths into components often leads to unexpected side effects, like security risks due to path traversal, errors due to prohibited characters, and many other issues.
`SOIL` wants to prevent these errors by being explicit about the concept of a path.

At the same time, the concept allows users to create their own rules about what acceptable paths are, e.g. to work well with other systems.
It's also possible to transmit paths in messages as a single string with some separator sequence, if a custom message encoding is used.

#### How do I prevent path collisions, when multiple components produce the same states?

This will be a common consideration in more complex systems.
It's **not** recommended to give every component a unique path prefix, so that values end up in a separate sub-tree of the data.
You may instantiate your components with a connection to a configuration storage, where they might get the sub-path to use for their data.

Another option is to consider each component like a storage device in Linux, which can be mounted in a file system in any directory, not just the root.
Write small "adapters" in your system that interface with components, and redistribute the data in a different structure: A sensor could just publish a `temperature` state, which get's converted to `galley.temperature` by another component, and is finally put in a database as `habitat.galley.temperature`.
Each layer adds context, which the contained primitive components don't need to know about.

#### Why are `Integer` and `Double` not combined into a `Numeric` type?

It could be argued that an integer is just a special case of a decimal value that only allows natural numbers.
But in practice, almost all systems differentiate between the two, and there are issues with precision and value ranges when mixing them. SOIL therefore distinguishes them as well.
But feel free to only use `Double` states.

#### Why are there no integer and double sizes for the primitive types, like `Int32`?

Value ranges are mostly an implementation detail, either for data transmission or storage, and don't really have meaning for an abstract state.
Of course states can have allowed ranges, limits, or other restrictions, but these are better expressed as separate states: The state `sensor.temperature` could have children `sensor.temperature.range.minimum` and `sensor.temperature.range.maximum`.
These structures are considered mission-specific conventions, and are out of scope for SOIL.

#### Why are booleans not a primitive type?

There are two reasons. Booleans are just integers with a very small value range (0 to 1), which is also how they are treated in some programming languages like Python.
Value ranges are most often an implementation detail of the transmission protocol, and are not inherently connected to the state itself.
Additionally, boolean values are mostly treated as enumerations, where `false` indicates one state (off, disabled, unavailable, etc.) while `true` indicates another.
It's recommended to either use human-readable strings, which are required for displays anyways, or to just use an integer (which can also easily be extended with more cases in the future).

#### How does a component advertise commanding possibilities?

A component can subcribe to specific states via subscription requests.
A subscription indicates "I'm interested in this data, and I will act on it", which is precicely what commands describe as well.
In many systems you will also know before deployment which inputs are possible, e.g. through documentation.

#### How does a component indicate the outputs it can provide?

Components can be queried for the states they contain.
The structure can be retrieved via request messages.
If you want the full tree, simply send a request for the root path (without a `depth` indicator), and the component will respond with the full tree, including all current values.
You can also discover the tree iteratively by restricting the depth of the response or querying subpaths.

#### Shouldn't a component respond to a subscription with the current values?

That's a mechanism that was considered, but it was decided against it.
- It adds more complexity for whole protocol
- A component may want to construct more elaborate subscriptions without receiving many unnecessary values
- Current values can be requested separately
