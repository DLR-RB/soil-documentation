#  Subscriptions

The concept of requesting data of interest from components can be described as "hierarchical subscriptions with scoped exclusions".
Components in the network can subscribe to paths in order to receive updates when the data changes.
Additionally, clients can unsubscribe from paths, to exclude specific updates from existing subscriptions.
With this mechanism, components can build complex include/exclude rules to manage the updates they want to receive.

 > For brevity, all paths in this document are abbreviated using `.`, e.g. `a.b` represents the path `["a", "b"]`.

## Creating subscriptions

Components can subscribe to sub-trees of the data, based on the paths.
A subscription to the path `a.b` will provide the component with updates to this path, as well as all paths contained within `a.b`, like `a.b.c`, or `a.b.e.f`.
A subscriber will not receive updates outside of the subscription, e.g. for `a`, `a.c`, or `a.c.b`.

### Subscription depth

It's possible to specify a `depth` of the subscription, which will limit the number of path components below the subscribed path that are considered.
A component subscribing to `a` with `depth = 1` will receive updates to `a`, `a.b`, and `a.c`, but not for `a.b.c`.
A depth of `0` will only provide updates for the specified path, but no sub-elements of it.
If no depth is specified, then all updates to the sub-tree of the path will be published.

### Ignoring updates (exclusions)

In addition to subscribing for updates, it's also possible to ignore sub-trees within a subscription.
The mechanism works similar to subscriptions with an optional `depth`, but limits existing subscriptions.
If a component subscribes to `a`, but excludes `a.b`, then it will receive updates for `a`, and `a.c`, but not for `a.b` or `a.b.c`.

### Composing subscriptions

Combining subscriptions and exclusions allows the creation of complex requests tailored to the needs of the component, while minimizing data transfer and processing.

## Update applicability

The subscriptions and exclusions are used to determine whether an update to a path is of interest to a component.

This process follows some simple rules:
- Exclusions override subscriptions (subscribing to `a.b` and excluding `a.b` will exclude it)
- The path is traversed from bottom to top (evaluting `a.b.c` before `a.b` before `a`)
- Lower-level rules override higher level rules (an exclusion for `a.b` overrides a subscription for `a`)
- Newer rules override older rules for the same path and action (e.g. subscribing to `a.b` with `depth = 1` after subscribing with `depth = 2`)
- Subscriptions and exclusions can exist for the same path, if they have different depths (e.g. subscribing to all of `a`, but excluding `a` with `depth = 1`)