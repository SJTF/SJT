# SJT (Structured JSON Table) Specification

### Overview

Structured JSON Table (SJT) is a lightweight, schema-based data encoding format that compresses hierarchical JSON-like data by separating structure (headers) from values. It is optimized for repetitive structures, such as arrays of objects, enabling faster parsing and significantly reduced size compared to raw JSON.

---

## Table of Contents

* [Motivation](#motivation)
* [Data Model](#data-model)
* [Supported Types](#supported-types)
* [Header Format](#header-format)
* [Data Representation (`Data`)](#data-representation-data)
* [Examples](#examples)

  * [Object](#object)
  * [Array of Objects](#array-of-objects)
  * [Nested Object](#nested-object)
  * [Nested Array of Objects](#nested-array-of-objects)
* [Encoding/Decoding Rules](#encodingdecoding-rules)
* [Encoding Algorithm](#encoding-algorithm)
* [Decoding Algorithm](#decoding-algorithm)
* [Server Encoding Note](#server-encoding-note)
* [Constraints](#constraints)
* [Advantages](#advantages)
* [Benchmarks](#benchmarks)
* [Use Cases](#use-cases)
* [Appendix A — File Extension and Media Type Specification](#appendix-a--file-extension-and-media-type-specification)


---

### Motivation

Traditional JSON represents each object or array with its full property names, leading to redundant transmission and increased payload size. SJT separates structure and data, allowing the client and server to reconstruct objects without repeating field names.

---

## Data Model

An SJT document is an array with two elements:

```js
[ header, values ]
```

* `header`: A structure descriptor representing the schema of the input data.
* `values`: A flattened array containing the actual data values that conform to the header structure.

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

## Header Format

The header is recursively structured to mirror the shape of the data. It uses a combination of:

* Primitive keys: strings denoting the names of properties.
* Nested descriptors: `[key, descriptor]` pairs for nested objects or arrays.
* `null`: used to represent a flat primitive array (e.g., `[1, 2, 3]`).

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

## Encoding Algorithm

### 1. `extractHeader(data)`

Returns the recursive header of the given object or array.

* If input is an array of primitives → return `[null]`
* If input is an array of objects → recurse on the first element's structure
* If input is an object:

  * For each key:

    * If the value is primitive → add key
    * If value is object or array → recurse and store as `[key, childHeader]`

### 2. `getValues(header, data)`

Returns the values in the order defined by the `header`.

* Traverses recursively with the same logic as `extractHeader`, pushing values in the same order.

### 3. `encodeSJT(data)`

Combines both header and values:

```js
const header = extractHeader(data);
const values = getValues(header, data);
return [header, values];
```

---

### Decoding Algorithm

The decoder reconstructs the original structure by walking the header and reading the corresponding values in sequence.

```ts
function decodeSJT([header, values]): any {
  // internal recursive function to walk the header and map values back
}
```

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
* `[null]` in header indicates a primitive array
* Header always maps directly to data layout
* All data arrays are wrapped once, even for primitive values
* Mixed types in primitive arrays are discouraged and not supported

---

### Constraints

* The structure must be **uniform** across all entries.
* Header keys must be **valid JSON object keys**.
* Mixed-type fields or varying structures across entries are **not supported**.

---

### Advantages

* **Compression:** Smaller output size than JSON, especially for structured data.
* **Speed:** Fast encode/decode; `decodeSJT` often outperforms `JSON.parse`.
* **Simplicity:** Pure JSON-compatible structure, no custom binary format.
* **Schema Extraction:** Header provides a lightweight, self-contained schema.

---

## Benchmarks

| Format         | Size (KB) | Encode Time | Decode Time |
|----------------|-----------|-------------|-------------|
| JSON           | 3849.34   | 41.81 ms    | 51.86 ms    |
| JSON + Gzip    | 379.67    | 55.66 ms    | 39.61 ms    |
| MessagePack    | 2858.83   | 51.66 ms    | 74.53 ms    |
| SJT (json)     | 2433.38   | 36.76 ms    | 42.13 ms    |
| SJT + Gzip     | 359.00    | 69.59 ms    | 46.82 ms    |


---

## Appendix A — File Extension and Media Type Specification

### A.1 File Extensions

To facilitate interoperability, file recognition, and tool support, the following **file extensions** are designated for SJT (Structured JSON Table) data:

| Extension   | Description                                 | Usage Notes                                  |
| ----------- | ------------------------------------------- | -------------------------------------------- |
| `.sjt`      | Raw SJT format (uncompressed)               | Primary standard for SJT files               |
| `.sjz`      | Gzipped SJT (`SJT + gzip`)                  | Recommended for network transmission         |
| `.sjt.json` | JSON-compatible SJT variant                 | Transitional use in legacy JSON environments |
| `.sjt.gz`   | Alternate to `.sjz` with explicit gzip hint | Useful for systems that rely on `.gz` suffix |

> **Recommendation:** Use `.sjt` for uncompressed data, and `.sjz` for compressed data.

### A.2 Media (MIME) Type

To support content negotiation and correct handling over HTTP or similar protocols, the following **media type** is proposed for SJT:

```
Content-Type: application/vnd.sjt+json
```

* This type denotes that the payload conforms to the **SJT format**, which is a structured, compressed variant of JSON.
* When used in compressed form (e.g., `gzip`), the following header SHOULD be included:

```
Content-Encoding: gzip
```

### A.3 Rationale

* Namespaced media type (`vnd.sjt+json`) ensures clear distinction from ordinary JSON while retaining JSON compatibility.
* Standardized extensions improve editor recognition, versioning workflows, and interoperability across tooling.
* Explicit `.gz` suffixes (e.g., `.sjz`, `.sjt.gz`) enable seamless integration with existing decompression tools and pipelines.

### A.4 Implementation Notes

* Applications MAY choose to detect SJT format via magic bytes or schema markers, but using correct file extensions and media types is STRONGLY RECOMMENDED.
* When embedding SJT in other containers (e.g., tarballs or archives), extensions SHOULD be preserved for extraction clarity.

---

---

**Version**: 1.0

**Author**: Yuki Akai

**Date**: 2025-07-25
