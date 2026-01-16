# Access control

`SOIL` is capable of simple but powerful access control mechanisms due to the tree structure of the data.

The main philosophy of access restrictions is:

> Restricted areas should be invisible to the client

This means that all functions should behave as if the restricted data is simply not present.
This is true for all read operations, although it can't be fully implemented for write operations.

## Approach

Essentially, a component can determine how to interact with other components by checking if they are allowed to read or write specific paths.
The access control logic can therefore be covered by only two functions:

```
canRead(component: ComponentId, path: Path) -> Bool
canWrite(component: ComponentId, path: Path) -> Bool
```

These functions can be used for all interactions to determine appropriate behaviour.
A suggestion for more complex scenarios can be found [below](#complex-access-api).

## Incoming values

A component that receives a [value message](Messages.md#value-message) calls `canRead()` with the component id (determined by the transport layer) and the `path` within the message.
If the result is `true`, then the sample can be saved (or forwarded).
If the write request is rejected, then the component discards the message.

If the received message contains a `request` id, then an acknowledgement is sent as if the data was saved (in accordance with the main philosophy above).
Note that it's not possible to fully hide the missing write permissions from other components:
They can request historic values to check if their data was actually stored.
This could only be prevented by saving all illegitimate writes separately, and provide them to the restricted component if needed.
This would be a major effort with an increased attack surface for denial-of-service attacks, so it is not recommended in practice.

It's also possible to implement clients that only save received values if they are in response to a request.

> The `canRead` function may be extended with a `wasRequested` argument.

## Value requests

For received requests for values, the component should call `canRead()` for all paths that are part of the response,
and limit the transmitted values to those that are allowed.
If the restrictions cause the responses to be empty, and a `request` id was set, then the component should respond with an empty message of the requested root path.

## Subscriptions

Subscriptions for components should be accepted independent of the access rights of the component.
This allows subscriptions to be established, and access to resources can be granted later.
It also simplifies the reasoning about the data to deliver, since components can simply subscribe to the full tree,
and then receive only the data of paths where they have access.

Due to this mechanism, a component needs to check the access manager before sending data to subscribers.

## Exclusions

There are also no access rights involved when requesting exclusions.

## History requests

Read access to the requested path is checked before the data is gathered.
Components may introduce more fine-grained checks to limit the history, e.g. only provide the last 24 hours.
The restricted interval should then be read from the database, and provided it as usual.

## Complex access API

To cover more complex access rules a full API may look like this:

```
canWrite(component: ComponentId, path: Path, time: Timestamp, wasRequested: Bool) -> Bool
canRead(component: ComponentId, path: Path, time: Timestamp) -> Bool
canReadHistory(component: ComponentId, path: Path, start: Timestamp, end: Timestamp) -> Range<Timestamp>?
```