# Construction and Conversion Methods

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/cpumask/blob/a7cfa639/README.md)
> * [src/lib.rs](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs)

This page documents all methods for creating `CpuMask` instances and converting between different representations. It covers the complete set of constructors, factory methods, and type conversion utilities provided by the `CpuMask<SIZE>` struct.

For information about querying and inspecting existing `CpuMask` instances, see [Query and Inspection Operations](/arceos-org/cpumask/2.2-query-and-inspection-operations). For modifying `CpuMask` state after construction, see [Modification and Iteration](/arceos-org/cpumask/2.3-modification-and-iteration).

## Construction Methods

The `CpuMask<SIZE>` struct provides multiple construction patterns to accommodate different use cases and data sources.

### Basic Construction

The most fundamental construction methods create `CpuMask` instances with predefined bit patterns:

|Method|Purpose|Bit Pattern|
| --- | --- | --- |
|new()|Empty cpumask|All bits set tofalse|
|full()|Complete cpumask|All bits set totrue|
|mask(bits: usize)|Range-based mask|Firstbitsindices set totrue|

The `new()` method [src/lib.rs(L74 - L76)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L74-L76) creates an empty cpumask by delegating to the `Default` trait implementation. The `full()` method [src/lib.rs(L79 - L84)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L79-L84) creates a cpumask with all valid bits set using the underlying `Bitmap::mask(SIZE)` operation. The `mask(bits: usize)` method [src/lib.rs(L89 - L94)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L89-L94) creates a mask where indices `0` through `bits-1` are set to `true`.

### Data-Driven Construction

For scenarios where `CpuMask` instances must be created from existing data, several factory methods accept different input formats:

The `from_value(data)` method [src/lib.rs(L97 - L102)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L97-L102) accepts data of the backing store type, which varies based on the `SIZE` parameter due to storage optimization. The `from_raw_bits(value: usize)` method [src/lib.rs(L106 - L119)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L106-L119) constructs a cpumask from a raw `usize` value, with assertion checking to ensure the value fits within the specified size. The `one_shot(index: usize)` method [src/lib.rs(L123 - L128)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L123-L128) creates a cpumask with exactly one bit set at the specified index.

### Construction Method Flow

```mermaid
flowchart TD
A["CpuMask Construction"]
B["new()"]
C["full()"]
D["mask(bits)"]
E["from_value(data)"]
F["from_raw_bits(value)"]
G["one_shot(index)"]
H["From trait implementations"]
I["Bitmap::new()"]
J["Bitmap::mask(SIZE)"]
K["Bitmap::mask(bits)"]
L["Bitmap::from_value(data)"]
M["Manual bit setting loop"]
N["Bitmap::new() + set(index, true)"]
O["Array to Bitmap conversion"]
P["CpuMask { value: bitmap }"]

A --> B
A --> C
A --> D
A --> E
A --> F
A --> G
A --> H
B --> I
C --> J
D --> K
E --> L
F --> M
G --> N
H --> O
I --> P
J --> P
K --> P
L --> P
M --> P
N --> P
O --> P
```

Sources: [src/lib.rs(L74 - L128)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L74-L128)

### Array-Based Construction

For larger cpumasks that use array backing storage, the library provides `From` trait implementations for `u128` arrays of various sizes. These implementations support cpumasks with sizes 256, 384, 512, 640, 768, 896, and 1024 bits:

```mermaid
flowchart TD
A["[u128; 2]"]
B["CpuMask<256>"]
C["[u128; 3]"]
D["CpuMask<384>"]
E["[u128; 4]"]
F["CpuMask<512>"]
G["[u128; 5]"]
H["CpuMask<640>"]
I["[u128; 6]"]
J["CpuMask<768>"]
K["[u128; 7]"]
L["CpuMask<896>"]
M["[u128; 8]"]
N["CpuMask<1024>"]
O["cpumask.into()"]

A --> B
B --> O
C --> D
D --> O
E --> F
F --> O
G --> H
H --> O
I --> J
J --> O
K --> L
L --> O
M --> N
N --> O
```

Sources: [src/lib.rs(L328 - L368)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L328-L368)

## Conversion Methods

The `CpuMask` struct provides multiple methods for extracting data in different formats, enabling interoperability with various system interfaces and storage requirements.

### Value Extraction

The primary conversion methods expose the underlying storage in different forms:

The `into_value(self)` method [src/lib.rs(L131 - L134)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L131-L134) consumes the cpumask and returns the backing store value. The `as_value(&self)` method [src/lib.rs(L137 - L140)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L137-L140) provides a reference to the backing store without transferring ownership. The `as_bytes(&self)` method [src/lib.rs(L143 - L146)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L143-L146) returns a byte slice representation of the cpumask data.

### Bidirectional Array Conversion

The library implements bidirectional conversion between `CpuMask` instances and `u128` arrays, enabling seamless integration with external APIs that expect array representations:

```mermaid
flowchart TD
A["CpuMask<256>"]
B["[u128; 2]"]
C["CpuMask<384>"]
D["[u128; 3]"]
E["CpuMask<512>"]
F["[u128; 4]"]
G["CpuMask<640>"]
H["[u128; 5]"]
I["CpuMask<768>"]
J["[u128; 6]"]
K["CpuMask<896>"]
L["[u128; 7]"]
M["CpuMask<1024>"]
N["[u128; 8]"]

A --> B
B --> A
C --> D
D --> C
E --> F
F --> E
G --> H
H --> G
I --> J
J --> I
K --> L
L --> K
M --> N
N --> M
```

Sources: [src/lib.rs(L370 - L410)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L370-L410)

### Storage Type Relationships

The conversion methods interact with the automatic storage optimization system, where the backing store type depends on the `SIZE` parameter:

```mermaid
flowchart TD
A["SIZE Parameter"]
B["Storage Selection"]
C["bool"]
D["u8"]
E["u16"]
F["u32"]
G["u64"]
H["u128"]
I["[u128; N]"]
J["from_value(bool)"]
K["from_value(u8)"]
L["from_value(u16)"]
M["from_value(u32)"]
N["from_value(u64)"]
O["from_value(u128)"]
P["from_value([u128; N])"]
Q["CpuMask<1>"]
R["CpuMask"]
S["CpuMask"]

A --> B
B --> C
B --> D
B --> E
B --> F
B --> G
B --> H
B --> I
C --> J
D --> K
E --> L
F --> M
G --> N
H --> O
I --> P
J --> Q
K --> R
L --> R
M --> R
N --> R
O --> R
P --> S
```

Sources: [src/lib.rs(L97 - L102)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L97-L102) [src/lib.rs(L12 - L16)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L12-L16)

## Type Safety and Validation

Construction methods incorporate compile-time and runtime validation to ensure cpumask integrity:

* The `mask(bits)` method includes a debug assertion [src/lib.rs(L90)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L90-L90) that `bits <= SIZE`
* The `from_raw_bits(value)` method validates [src/lib.rs(L107)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L107-L107) that `value >> SIZE == 0`
* The `one_shot(index)` method asserts [src/lib.rs(L124)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L124-L124) that `index < SIZE`
* Array-based `From` implementations use const generics to enforce size compatibility at compile time

These validation mechanisms prevent buffer overflows and ensure that cpumask operations remain within the defined size boundaries.

Sources: [src/lib.rs(L90)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L90-L90) [src/lib.rs(L107)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L107-L107) [src/lib.rs(L124)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L124-L124) [src/lib.rs(L328 - L410)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L328-L410)