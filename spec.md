**Title**: Structured JSON Table Format (SJTF) Specification

**Version**: 1.0

**Author**: Yuki Akai

**Date**: 2025-07-25

---

### Overview

Structured JSON Table Format (SJTF) is a compact and bandwidth-efficient data representation designed to encode large, structured datasets—especially those with uniform shape or schema—into a nested, index-driven format. This reduces redundancy and improves serialization/deserialization speed over conventional JSON.

---

### Motivation

Traditional JSON represents each object or array with its full property names, leading to redundant transmission and increased payload size. SJTF separates structure and data, allowing the client and server to reconstruct objects without repeating field names.

---

### Format Structure

The SJTF payload is a 2-element array:

```
[ <Header>, <Data> ]
```

* **Header**: A recursive array of strings and nested arrays representing the schema.
* **Data**: An array of records, each record maps to a header schema by position.

#### Example

```json
[
  ["id", ["user", ["name", "age"]]],
  [
    [1, [["Yuki", 24]]],
    [2, [["Aki", 24]]]
  ]
]
```

Which decodes into:

```json
[
  {
    "id": 1,
    "user": {
      "name": "Yuki",
      "age": 24
    }
  },
  {
    "id": 2,
    "user": {
      "name": "Aki",
      "age": 24
    }
  }
]
```

---

### Parsing Algorithm (Client-side)

To decode:

1. Parse the header recursively to build a template.
2. For each entry in the data array, recursively apply the template to assign keys.

> The full technical specification implementation of SJTF in JavaScript/TypeScript is available here.

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

---

### Use Cases

* APIs with large paginated results (e.g., Discord, Google).
* Event logs or metrics.
* Realtime streaming of structured data.
