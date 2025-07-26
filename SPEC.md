# SJT (Structured JSON Table) Specification

### Overview

Structured JSON Table (SJT) is a compact, tree-structured encoding format for JSON-like, bandwidth-efficient data representation designed to encode large, structured datasets—especially those with uniform shape or schema—into a nested, index-driven format. This reduces redundancy and improves serialization/deserialization speed over conventional JSON.

---

## Table of Contents

* [Motivation](#motivation)
* [Encoding Overview](#encoding-overview)
* [Supported Types](#supported-types)
* [Schema Definition (`Header`)](#schema-definition-header)
* [Data Representation (`Data`)](#data-representation-data)
* [Examples](#examples)

  * [Object](#object)
  * [Array of Objects](#array-of-objects)
  * [Nested Object](#nested-object)
  * [Nested Array of Objects](#nested-array-of-objects)
* [Encoding/Decoding Rules](#encodingdecoding-rules)
* [Encoding Algorithm](#encoding-algorithm-server-side)
* [Decoding Algorithm (Client-side)](#decoding-algorithm-client-side)
* [Server Encoding Note](#server-encoding-note)
* [Constraints](#constraints)
* [Advantages](#advantages)
* [Use Cases](#use-cases)

---

### Motivation

Traditional JSON represents each object or array with its full property names, leading to redundant transmission and increased payload size. SJT separates structure and data, allowing the client and server to reconstruct objects without repeating field names.

---

## Encoding Overview

Every SJT-encoded data consists of two top-level parts:

```ts
[
  <header: SjtHeader>,
  <data: SjtData>
]
```

Where:

* `header`: Defines structure and keys.
* `data`: Actual values to be deserialized following the header schema.
---

---

## Supported Types

* `null`
* `boolean`
* `number`
* `string`
* `array`
* `object`

Special handling is provided for arrays of objects.

---

## Schema Definition (`Header`)

The header is a recursive array that defines the structure of the data:

### Primitives / Flat Object

```ts
['id', 'name'] // => object { id: ..., name: ... }
[null] // => Primitive Array

```

### Array of Object

```ts
[['id', 'name']] // => array of object [{ id: ..., name: ... }, ...]
```

### Nested Object

```ts
['user', ['id', 'name']] // => { user: { id: ..., name: ... } }
```

### Nested Array of Object

```ts
['group', [['id', 'name']]] // => { group: [{ id: ..., name: ... }] }
```

Header elements can recurse indefinitely.

---

## Data Representation (`Data`)

The `data` field is an array of values matching the structure described by the `header`.

---

## Examples

### Object

```ts
Input:
{ id: 1, name: 'Yuki' }

Encoded:
header: ['id', 'name']
data: [1, 'Yuki']
```

### Array of Objects

```ts
Input:
[
  { id: 1, name: 'Yuki' },
  { id: 2, name: 'Aki' }
]

Encoded:
header: [['id', 'name']]
data: [
  [1, 'Yuki'],
  [2, 'Aki']
]
```

### Primitive Array

```ts
Input:
[
  1,2
]

Encoded:
header: [null]
data: [
  [1,2]
]

```

### Field containing a primitive array

```ts
Input:
{
 tag: ['ts', 'code']
}

Encoded:
header: ["tag", [null]]
data: [
  ['ts', 'code']
]

```



### Nested Object

```ts
Input:
{ user: { id: 1, name: 'Yuki' } }

Encoded:
header: ['user', ['id', 'name']]
data: [ [1, 'Yuki'] ]
```

### Nested Array of Objects

```ts
Input:
{
  message: 'hello',
  users: [
    { id: '1', name: 'Yuki' },
    { id: '2', name: 'Aki' }
  ]
}

Encoded:
header: ['message', ['users', [['id', 'name']]]]
data: ['hello', [ ['1', 'Yuki'], ['2', 'Aki'] ]]
```

---

## Encoding/Decoding Rules

### Disambiguation:

*Header:*
* `['id', 'name']` → flat object
* `[['id', 'name']]` → array of objects
* `[null]` → primitive array

[]
Decoder can distinguish them via:

```ts
Array.isArray(header[0])
```

* `true` → array object mode
* `false` → plain object

### Nesting:

Headers can recurse to arbitrary depth:

```ts
['a', ['b', ['c', 'd']]] // { a: { b: { c: ..., d: ... } } }
```

### Deep Array of Object Nesting

```ts
['a', [['b', [['c', 'd']]]]]
// { a: [ { b: [ { c: ..., d: ... } ] } ] }
```

This allows encoding deeply nested array/object combinations naturally.


---

### Encoding Algorithm (Server-side)

#### Step 1: Validate Input

* Input must be an uniformly structured JSON objects.

#### Step 2: Build Header

* Recursively walk through object to collect all keys.
* If a value is a nested object or array of structured objects, recurse and build a nested header.

#### Step 3: Encode Data

* For each object in the array:

  * Extract values in the order and shape defined by the header.
  * Use recursion to navigate into nested structures.

#### Step 4: Output

* Return the final structure:

```ts
[
  Header, // The field template
  Data[]  // Flattened object values in header order
]
```

---

### Decoding Algorithm (Client-side)

#### Step 1: Read Header and Data

* Extract header and data entries from the 2-element array.

#### Step 2: Rebuild Objects

* For each entry in the data array:

  * Recursively apply keys from the header to inject values into new objects.

#### Step 3: Output

* Return an array of fully reconstructed JSON objects.

---

## Server Encoding Note

When encoding on the server:

* You must traverse your source JSON and extract keys in the exact recursive order.
* Always represent array of objects as nested headers: `[['key1', 'key2']]`
* For deeply nested structures, walk the structure depth-first and wrap arrays of objects accordingly.
* Always ensure `data` mirrors the `header` shape, in both structure and order.

---

### Constraints

* The structure must be **uniform** across all entries.
* Header keys must be **valid JSON object keys**.
* Mixed-type fields or varying structures across entries are **not supported** in this version.

---

### Advantages

* Minimizes repeated keys → smaller payload size.
* Easy to cache structure separately.
* Fast deserialization with uniform template.
* Works well with JSON-compatible transport.
* Ideal for tabular or API responses with fixed schema.

---

### Use Cases

* APIs with large paginated results (e.g., Discord, Google).
* Event logs or metrics.
* Realtime streaming of structured data.

---

**Version**: 1.0

**Author**: Yuki Akai

**Date**: 2025-07-25
