
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

#### Input

#### Logic

#### Value

#### Timestamp

####
