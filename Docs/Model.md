# SOIL Data Model

This document describes the SOIL data model.

Data in SOIL is structured around [Values](#value), [Samples](#sample), and [SampleTrees](#sample-tree).

## Value

A SOIL `Value` is a piece of data that has one of the following types:

| Type       | Description                                                                                             |
|:-----------|:--------------------------------------------------------------------------------------------------------|
| `Null`     | The absence of a value                                                                                  |
| `Integer`  | An integer value, can be negative. The supported range is implementation-specific                       |
| `Double`   | A floating-point value. The supported range and precision is implementation-specific                    |
| `String`   | A UTF-8 string value. The supported length is implementation-specific                                   |
| `Data`     | Binary data. The supported length is implementation-specific                                            |
| `Path`     | A list of strings representing a SOIL path                                                              |
| `List`     | A list of `Value`s. Each element of the list can be an arbitrary value, including nested lists and maps |
| `Map`      | A list of `Value`s keyed by strings. Each element associated with a key can be an arbitrary value       |

The recursive structure with lists and maps allows values to represent arbitrarily complex data structures.

## Sample

A sample is the combination of a [Value](#value) with a timestamp.

| Field     | Type             | Description                                                                       |
|:----------|:-----------------|:----------------------------------------------------------------------------------|
| `value`   | `Value`          | The value associated with the sample.                                             |
| `time`    | `Optional<Date>` | The timestamp associated with the `value`. An empty time implies the current time |

The exact implementation and reference date of a timestamp is not defined by SOIL, and is chosen by the implementation or application.

## Sample Tree

A sample tree is a recursive structure that contains samples, and is organized as a data tree.

| Field      | Type                                | Description                                                                 |
|:-----------|:------------------------------------|:----------------------------------------------------------------------------|
| `samples`  | `Array<Sample>` | The samples at this node (may be empty)                                                         |
| `children` | `Map<String, SampleTree>` | The child nodes below this node, indexed by their name (path component), may be empty |

Each child is again defined as a `SampleTree`, so structures with an arbitrary number of levels are possible.
