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
* [Error Handling](#error-handling)
* [Server Encoding Note](#server-encoding-note)
* [Constraints](#constraints)
* [Advantages](#advantages)
* [Benchmarks](#benchmarks)
* [Use Cases](#use-cases)
* [Appendix A — File Extension and Media Type Specification](#appendix-a--file-extension-and-media-type-specification)
* [Appendix B: Header Grammar & JSON Schema Mapping](#appendix-b--header-grammar---json-schema-mapping)


---

### Motivation

Traditional JSON represents each object or array with its full property names, leading to redundant transmission and increased payload size. SJT separates structure and data, allowing the client and server to reconstruct objects without repeating field names.

---

## Data Model

An SJT document is an array with two elements:

```ts
[
  header: SjtHeader,
  data: SjtData,
  SjtMetadata?  // optional
]
```

* `header`: A structure descriptor representing the schema of the input data.
* `data`: A flattened array containing the actual data values that conform to the header structure.

> Note: While the outer structure is a standard JSON array, it must **not** be interpreted as a generic array. This format is governed by the SJT specification, where:
>
> * The **first element** must always be a valid `SjtHeader`
> * The **second element** must always be a valid `SjtData` block
* The first two items (`header` and `data`) are mandatory and define the structure and values.
* The third item (`metadata`) is optional and **reserved for future extensions** (see appendix).

  * It must be an object if present.
  * Parsers **must not fail** if `metadata` is present but unrecognized.
  * This allows forward-compatible extensions (e.g., `version`, `checksum`, etc.).

---

###  Definitions

#### `SjtHeader`

Defines the fields, their order, and metadata.

The `header` field in SJT is always an **array**, possibly empty array, where each element (if present) conforms to the recursive `Header` structure defined as:

```ts
type SjtHeader =
  | string
  | null
  | [string, Header[]]           // Nested object key
  | Header[]                     // Object array
```
This allows representing an empty object with:

```json
header = []
body   = []
```

Which decodes to:

```json
{}
```

If the array is not empty, each element must conform to the `Header` grammar above.

In other words, the top-level `header` is always of type `SjtHeader[]`.

#### `SjtData`

Contains the actual content, organized in **column-major layout**

The `Data` field in SJT is always an **array** containing one or more elements, where each element conforms to the recursive `Data` structure defined as:

```ts
SjtData = string | boolean | null | number | SjtData[]   // Each inner array is a column of values
```

In other words, the top-level `Data` is always of type `SjtData[]`.

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

---

### **Rule: `null` Must Be a Standalone Header**

In SJT, the special value `null` in the `header` array is used to indicate that the corresponding column in `data` is a **primitive array** (see previous section). However, to avoid ambiguity, the following strict rule applies:

> **A `null` entry must be the **only** item in its `header` array.**

#### Valid Header for Primitive Array:

```jsonc
[ null ]              // means: primitive array, e.g. [1, 2, 3]
```

#### Invalid Headers:

```jsonc
[ null, null ]        // ❌ ambiguous: interpreted as an object with duplicate keys (invalid)
[ "name", null ]      // ❌ mixed keys in same object (invalid if used to define object structure)
```

Including multiple `null` values within the same array creates ambiguity, as SJT treats header arrays as **structured key paths** for object construction. Having duplicate keys (`null`, `null`) is structurally invalid and MUST result in a decoding error.

#### Rationale:

* SJT maps `header` to structured objects by aligning the shape of the header array with the shape of the `data`.
* `[null, null]` would imply an object with two identical `null` keys, which violates JSON object key uniqueness.

#### Enforcement:

If a `header` array contains more than one item and includes `null`, decoding MUST fail with an error.

---

### **Distinction Between `null` and `"null"` in Header Keys**

In JSON, all object keys must be strings. Therefore, to encode a key named `"null"` (the literal string), it must be explicitly written as a string:

#### Valid Representation of a `"null"` Key (string):

```jsonc
[ "null" ]         // means: [{ "null": value }]
[null]            // means: primitive array, NOT an object key
```

#### Invalid Representation:

```jsonc
["name", null] // Invalid
```

#### Rule:

> When encoding a key named `"null"`, use the string `'null'`.
> The bare value `null` in a header is reserved **only** for marking primitive arrays.

This distinction ensures clarity and avoids structural ambiguity when parsing and serializing SJT.

---

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

## Error Handling

Implementations of the SJT decoder **must strictly validate input structure** and **must throw an error** (or terminate with failure) upon encountering any of the following violations:

#### **Structural Violations**

* The root value is not an array of length 2 or 3
* The first element (header) is not an array
* The second element (data) is not an array

Header must conform to:

* A **string** (single column)
* An **array of**:

  * A `[string, array<header>]` (nested object)
  * A `array<header>` (array of structured object)

If not, throw:

```ts
throw new SJTInvalidHeaderError("Header structure is invalid. Expected string | [string, header[]] | header[][]");
```

#### **Semantic Violations**

* Rows or values contain undefined/unsupported JSON values (e.g., functions, `undefined`, symbols)

#### **`null` Header Behavior**

When a `header` entry is `null`, it indicates that the corresponding **data field must be one of the following**:

* A primitive value (`string`, `number`, `boolean`, or `null`)

Any other structure (such as objects or arrays containing objects) is considered invalid and must throw an error.

#### **Invalid JSON**

* The entire SJT input must be a valid JSON string or parsed JSON structure.
* If not parseable or `null`, throw:
  **`SJTParseError: Input is not valid JSON.`**

#### **Metadata Violations (non-fatal)**

* Metadata is optional; extra fields in metadata should be ignored or logged — not cause hard failure

---

**Decoder Behavior Summary**

| Error Type            | Must Throw? | Suggested Exception Name    |
| --------------------- | ----------- | --------------------------- |
| Invalid root shape    | ✅ Yes       | `SJTFormatError`            |
| Header mismatch       | ✅ Yes       | `SJTHeaderMismatchError`    |
| Data mismatch         | ✅ Yes       | `SJTDataMismatchError`      |
| Metadata irregularity | ❌ No        | (Optional: `SJTWarning`)    |

**Examples**:

```js

['name', ['user', ['id'] ]] // valid (nested object)

['name', ['user', 'id' ]] // Invalid

[null] // valid (primitive array)
[null, null] // Invalid
```

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

## Appendix B: Header Grammar & JSON Schema Mapping

#### Formal Recursive Header Grammar (EBNF Style)

```ebnf
header        ::= string
                | null                                      (*Primitive Array*)  
                | array of (string, array of header)       (* object with nested fields *)
                | array of array of header                 (* array of nested objects *)
data          ::= string | boolean | null | data[]
```

> `string`: Column name

> `null`: Column namePrimitive Array

> `[string, array<header>]`: Object with nested structure

> `array<array<header>>`: Array of structured objects

****
####  Valid Examples:

```js
["id", "name", "email"]                     //// Equivalent flat
[ "email", ["user", ["id", "name"]]]       // Nested object
[[ "email", ["user", ["id", "name"]]]]      // Array of nested object
```

---

### **B.1 Mapping to JSON Schema (for validation)**

Although SJT is structurally different from standard JSON, it can be described with a loose mapping to JSON Schema, especially for tools needing to validate SJT documents before decoding.

**Informal mapping**:

```jsonc
{
  "type": "array",
  "prefixItems": [
    {
      // Header section
      "type": "array",
      "items": {
        "type": "array",
        "items": {
          // Accept any value, stricter validation should be implemented in custom logic
          "type": ["string", "null", "array"]
        }
      }
      
    },
    {
      // Data section
      "type": "array",
      "items": {
        "type": "array",
        "items": {
          // Accept any value, stricter validation should be implemented in custom logic
          "type": ["string", "number", "boolean", "null", "array"]
        }
      }
    },
    {
      // Optional metadata
      "type": "object",
      "properties": {
        "version": { "type": "string" },
        "extensions": { "type": "object" }
      },
      "additionalProperties": true
    }
  ]
}
```

> ⚠️ This schema is for structural validation only. Type consistency and field alignment should be enforced via a separate SJT validator.

---

### **B.3 Metadata Field**

The third (optional) element of an SJT document can be a **metadata object**. It is reserved for versioning, extension registration, or runtime hints. It is **non-normative** for the core decoding but can be utilized by tooling.

**Example:**

```json
[
   ...
  ,
  {
    "version": "1.0",
    "generatedBy": "my-sjt-lib",
    "extensions": {
      "timestamp": "2025-07-25T13:00:00Z"
    }
  }
]
```

**Reserved keys:**

* `version`: The SJT format version (optional, informative)
* `extensions`: An object for any extended metadata
* Other keys may be used freely but **should not interfere with decoding logic**

---

---

**Version**: 1.0

**Author**: Yuki Akai

**Date**: 2025-07-25
