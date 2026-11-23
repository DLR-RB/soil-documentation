#  Subscriptions

The concept of requesting data of interest from components can be described as "hierarchical subscriptions with scoped exclusions".
Components in the network can subscribe to paths in order to receive updates when the data changes.
Additionally, clients can unsubscribe from paths, to exclude specific updates from existing subscriptions.
It's also possible to remove subscription rules.
With this mechanism, components can build complex include/exclude rules to manage the updates they want to receive.

 > For brevity, all paths in this document are abbreviated using `.`, e.g. `a.b` represents the path `["a", "b"]`.

## Creating subscriptions

Subscriptions are created by sending an `inclusion` or `exclusion` request to a component.

### Inclusions

Components can request to include sub-trees of the data, based on the paths.
An inclusion of the path `a.b` will provide the component with updates to this path, as well as all paths contained within `a.b`, like `a.b.c`, or `a.b.e.f`.
A subscriber will not receive updates outside of the inclusion, e.g. for `a`, `a.c`, or `a.c.b`.

### Exclusions

Conversely, exclusions specify which paths to ignore for the subscription.
The mechanism works similar to inclusions.
If a component includes `a`, but excludes `a.b`, then it will receive updates for `a`, and `a.c`, but not for `a.b` or `a.b.c`.

## Rule deletion

It's possible to delete subscription rules that are no longer needed, by sending a `clear` request.
A rule deletion request removes all inclusion and exclusion rules for the specified paths and depth.

### Subscription depth

It's possible to specify a `depth` for inclusions and exclusions, which will limit the number of path components below the specified path that are considered.
A component including `a` with `depth = 1` will receive updates to `a`, `a.b`, and `a.c`, but not for `a.b.c`.
A depth of `0` will only target the exact path, but no sub-elements of it.
If no depth is specified, then the whole sub-tree of the path will be considered.
The same rule applies for exlusions, and for removing rules.

### Composing subscriptions

Combining subscriptions and exclusions allows the creation of complex requests tailored to the needs of the component, while minimizing data transfer and processing.

Inclusions and exclusions can be combined according to some simple rules:
- The path is traversed from bottom to top (evaluting `a.b.c` before `a.b` before `a`)
- Lower-level rules override higher level rules (an exclusion for `a.b` overrides the inclusion of `a`)
- Shallower depths override higher depths (an inclusion of `a.b` with `depth = 1` overrides an exclusion with `depth = 2`)
- Exclusions override inclusions with the same depth (including `a.b` and excluding `a.b` will exclude it)

> Multiple inclusions and exclusions can exist for the same path (e.g. including to all of `a`, but excluding `a` with `depth = 1`)

## Responses to subscription requests

The component receiving the subscription and exclusion requests will only respond to the request if the `request` field is set,
in which case it sends an empty value message with the received path and request id.
Other messages are only sent when new data arrives.

The current values at the time of the subscription will not be transmitted automatically, but can be requested separately (note that exclusions are not considered when requesting trees).
