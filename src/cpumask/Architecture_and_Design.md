# Architecture and Design

> **Relevant source files**
> * [Cargo.toml](https://github.com/arceos-org/cpumask/blob/a7cfa639/Cargo.toml)
> * [src/lib.rs](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs)

This document covers the internal implementation details, storage optimization strategy, performance characteristics, and architectural decisions of the cpumask library. It explains how the `CpuMask` struct integrates with the `bitmaps` crate to provide efficient CPU affinity management. For API usage patterns and practical examples, see [Usage Guide and Examples](/arceos-org/cpumask/4-usage-guide-and-examples). For complete API documentation, see [API Reference](/arceos-org/cpumask/2-api-reference).

## Core Architecture

The cpumask library is built around a single primary type `CpuMask<const SIZE: usize>` that provides a type-safe, compile-time sized bitmap for CPU affinity management. The architecture leverages Rust's const generics and the `bitmaps` crate's automatic storage optimization to provide efficient memory usage across different CPU count scenarios.

### Primary Components

```mermaid
flowchart TD
subgraph subGraph2["Concrete Storage Types"]
    E["bool (SIZE=1)"]
    F["u8 (SIZE≤8)"]
    G["u16 (SIZE≤16)"]
    H["u32 (SIZE≤32)"]
    I["u64 (SIZE≤64)"]
    J["u128 (SIZE≤128)"]
    K["[u128; N] (SIZE>128)"]
end
subgraph subGraph1["Storage Abstraction"]
    B["Bitmap"]
    C["BitsImpl"]
    D["Bits trait"]
end
subgraph subGraph0["User API Layer"]
    A["CpuMask"]
end
L["IntoIterator"]
M["BitOps traits"]
N["Hash, PartialOrd, Ord"]

A --> B
A --> L
A --> M
A --> N
B --> C
C --> D
D --> E
D --> F
D --> G
D --> H
D --> I
D --> J
D --> K
```

**Core Architecture Design**

The `CpuMask` struct wraps a `Bitmap<SIZE>` from the bitmaps crate, which automatically selects the most efficient storage type based on the compile-time `SIZE` parameter. The `BitsImpl<SIZE>` type alias resolves to the appropriate concrete storage type, and the `Bits` trait provides uniform operations across all storage variants.

Sources: [src/lib.rs(L18 - L23)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L18-L23) [src/lib.rs(L7)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L7-L7) [src/lib.rs(L98 - L102)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L98-L102)

### Type System Integration

```mermaid
flowchart TD
subgraph subGraph2["Operational Traits"]
    H["BitAnd, BitOr, BitXor"]
    I["Not"]
    J["BitAndAssign, BitOrAssign, BitXorAssign"]
    K["IntoIterator"]
end
subgraph subGraph1["Trait Bounds"]
    C["Default"]
    D["Clone, Copy"]
    E["Eq, PartialEq"]
    F["Hash"]
    G["PartialOrd, Ord"]
end
subgraph subGraph0["Const Generic System"]
    A["const SIZE: usize"]
    B["BitsImpl: Bits"]
end

A --> B
B --> C
B --> D
B --> E
B --> F
B --> G
B --> H
B --> I
B --> J
B --> K
```

**Type System Design**

The library extensively uses Rust's trait system to provide consistent behavior across different storage types. All traits are conditionally implemented based on the capabilities of the underlying storage type, ensuring type safety while maintaining performance.

Sources: [src/lib.rs(L17)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L17-L17) [src/lib.rs(L25 - L66)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L25-L66) [src/lib.rs(L253 - L326)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L253-L326)

## Storage Optimization Strategy

The most significant architectural feature is the automatic storage optimization that occurs at compile time based on the `SIZE` parameter. This optimization is handled entirely by the `bitmaps` crate's `BitsImpl` type system.

### Storage Selection Logic

```mermaid
flowchart TD
A["Compile-time SIZE parameter"]
B["SIZE == 1?"]
C["bool storage1 bit"]
D["SIZE <= 8?"]
E["u8 storage8 bits"]
F["SIZE <= 16?"]
G["u16 storage16 bits"]
H["SIZE <= 32?"]
I["u32 storage32 bits"]
J["SIZE <= 64?"]
K["u64 storage64 bits"]
L["SIZE <= 128?"]
M["u128 storage128 bits"]
N["[u128; N] storageN = (SIZE + 127) / 128"]
O["Memory: 1 byte"]
P["Memory: 1 byte"]
Q["Memory: 2 bytes"]
R["Memory: 4 bytes"]
S["Memory: 8 bytes"]
T["Memory: 16 bytes"]
U["Memory: 16 * N bytes"]

A --> B
B --> C
B --> D
C --> O
D --> E
D --> F
E --> P
F --> G
F --> H
G --> Q
H --> I
H --> J
I --> R
J --> K
J --> L
K --> S
L --> M
L --> N
M --> T
N --> U
```

**Storage Selection Algorithm**

The storage selection is determined by the `BitsImpl<SIZE>` type alias from the bitmaps crate. This provides optimal memory usage by selecting the smallest storage type that can accommodate the required number of bits, with special handling for the single-bit case using `bool`.

Sources: [src/lib.rs(L12 - L16)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L12-L16) [src/lib.rs(L20)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L20-L20)

### Array Storage for Large Sizes

For CPU masks exceeding 128 bits, the library uses arrays of `u128` values. The implementation provides explicit conversion traits for common large sizes used in high-core-count systems.

|Size|Storage Type|Conversion Traits|
| --- | --- | --- |
|256|[u128; 2]|From<[u128; 2]>,Into<[u128; 2]>|
|384|[u128; 3]|From<[u128; 3]>,Into<[u128; 3]>|
|512|[u128; 4]|From<[u128; 4]>,Into<[u128; 4]>|
|640|[u128; 5]|From<[u128; 5]>,Into<[u128; 5]>|
|768|[u128; 6]|From<[u128; 6]>,Into<[u128; 6]>|
|896|[u128; 7]|From<[u128; 7]>,Into<[u128; 7]>|
|1024|[u128; 8]|From<[u128; 8]>,Into<[u128; 8]>|

Sources: [src/lib.rs(L328 - L410)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L328-L410)

## Integration with Bitmaps Crate

The cpumask library is designed as a domain-specific wrapper around the `bitmaps` crate, leveraging its efficient bitmap implementation while providing CPU-focused semantics and additional functionality.

### Delegation Pattern

```mermaid
flowchart TD
subgraph subGraph1["Bitmap Delegation"]
    I["Bitmap::new()"]
    J["Bitmap::mask(SIZE)"]
    K["Bitmap::mask(bits)"]
    L["Store::get()"]
    M["Bitmap::set()"]
    N["Bitmap::len()"]
    O["Store::first_index()"]
    P["Bitmap::invert()"]
end
subgraph subGraph0["CpuMask Methods"]
    A["new()"]
    B["full()"]
    C["mask(bits)"]
    D["get(index)"]
    E["set(index, value)"]
    F["len()"]
    G["first_index()"]
    H["invert()"]
end

A --> I
B --> J
C --> K
D --> L
E --> M
F --> N
G --> O
H --> P
```

**Method Delegation Architecture**

Most `CpuMask` methods directly delegate to the underlying `Bitmap<SIZE>` implementation, providing a thin wrapper that maintains the CPU-specific semantics while leveraging the optimized bitmap operations.

Sources: [src/lib.rs(L74 - L84)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L74-L84) [src/lib.rs(L167 - L234)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L167-L234)

### Storage Access Methods

The library provides multiple ways to access the underlying storage, enabling efficient integration with system APIs that expect raw bit representations:

```mermaid
flowchart TD
A["CpuMask"]
B["into_value()"]
C["as_value()"]
D["as_bytes()"]
E["from_value(data)"]
F["as Bits>::Store"]
G["& as Bits>::Store"]
H["&[u8]"]
I["CpuMask"]
J["Direct storage access"]
K["Borrowed storage access"]
L["Byte array representation"]
M["Construction from storage"]

A --> B
A --> C
A --> D
A --> E
B --> F
C --> G
D --> H
E --> I
F --> J
G --> K
H --> L
I --> M
```

**Storage Access Patterns**

These methods enable zero-copy conversions between `CpuMask` and its underlying storage representation, facilitating efficient interaction with system calls and external APIs that work with raw bit data.

Sources: [src/lib.rs(L131 - L146)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L131-L146)

## Performance Characteristics

The architectural design prioritizes performance through several key strategies:

### Operation Complexity

|Operation Category|Time Complexity|Space Complexity|Implementation|
| --- | --- | --- | --- |
|Construction|O(1)|O(1)|Direct storage initialization|
|Single bit access (get,set)|O(1)|O(1)|Direct bit manipulation|
|Bit counting (len)|O(log n)|O(1)|Hardware population count|
|Index finding (first_index,last_index)|O(log n)|O(1)|Hardware bit scan|
|Iteration|O(k)|O(1)|Where k = number of set bits|
|Bitwise operations|O(1)|O(1)|Single instruction for small sizes|

### Memory Layout Optimization

```mermaid
flowchart TD
subgraph subGraph1["Performance Impact"]
    E["Cache-friendly access"]
    F["Minimal memory overhead"]
    G["SIMD-compatible alignment"]
    H["Efficient copying"]
end
subgraph subGraph0["Memory Layout Examples"]
    A["CpuMask<4>Storage: u8Memory: 1 byteAlignment: 1"]
    B["CpuMask<16>Storage: u16Memory: 2 bytesAlignment: 2"]
    C["CpuMask<64>Storage: u64Memory: 8 bytesAlignment: 8"]
    D["CpuMask<256>Storage: [u128; 2]Memory: 32 bytesAlignment: 16"]
end

A --> E
B --> F
C --> G
D --> H
```

**Memory Efficiency Design**

The automatic storage selection ensures that CPU masks consume the minimum possible memory while maintaining optimal alignment for the target architecture. This reduces cache pressure and enables efficient copying and comparison operations.

Sources: [src/lib.rs(L12 - L16)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L12-L16)

## Iterator Implementation

The iterator design provides efficient traversal of set bits without allocating additional memory, supporting both forward and backward iteration.

### Iterator State Management

```mermaid
stateDiagram-v2
[*] --> Initial
Initial --> Forward : "next()"
Initial --> Backward : "next_back()"
Forward --> ForwardActive : "found index"
Forward --> Exhausted : "no more indices"
Backward --> BackwardActive : "found index"
Backward --> Exhausted : "no more indices"
ForwardActive --> Forward : "next()"
ForwardActive --> Collision : "head >= tail"
BackwardActive --> Backward : "next_back()"
BackwardActive --> Collision : "head >= tail"
Collision --> Exhausted
Exhausted --> [*]
```

**Iterator State Machine**

The `Iter<'a, SIZE>` struct maintains head and tail pointers to support bidirectional iteration while detecting when the iterators meet to avoid double-yielding indices.

Sources: [src/lib.rs(L428 - L518)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L428-L518)

### Iterator Performance

The iterator implementation leverages the underlying bitmap's efficient index-finding operations:

* `first_index()`: Hardware-accelerated bit scanning
* `next_index(index)`: Continues from previous position
* `prev_index(index)`: Reverse bit scanning
* Memory usage: Fixed 24 bytes (two `Option<usize>` + reference)

Sources: [src/lib.rs(L444 - L479)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L444-L479) [src/lib.rs(L486 - L517)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L486-L517)

## API Design Patterns

The library follows consistent design patterns that enhance usability and performance:

### Constructor Patterns

```mermaid
flowchart TD
subgraph subGraph1["Use Cases"]
    G["Empty initialization"]
    H["All CPUs available"]
    I["Range-based masks"]
    J["Single CPU selection"]
    K["Integer conversion"]
    L["Type-safe conversion"]
end
subgraph subGraph0["Construction Methods"]
    A["new()"]
    B["full()"]
    C["mask(bits)"]
    D["one_shot(index)"]
    E["from_raw_bits(value)"]
    F["from_value(data)"]
end

A --> G
B --> H
C --> I
D --> J
E --> K
F --> L
```

**Constructor Design Philosophy**

Each constructor serves a specific use case common in CPU affinity management, from empty masks for incremental building to full masks for restriction-based workflows.

Sources: [src/lib.rs(L72 - L128)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L72-L128)

### Error Handling Strategy

The library uses a combination of compile-time guarantees and runtime assertions to ensure correctness:

* **Compile-time**: `SIZE` parameter ensures storage adequacy
* **Debug assertions**: Index bounds checking in debug builds
* **Runtime panics**: Explicit validation for user-provided values

Sources: [src/lib.rs(L90)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L90-L90) [src/lib.rs(L107)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L107-L107) [src/lib.rs(L124)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L124-L124) [src/lib.rs(L169)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L169-L169) [src/lib.rs(L178)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L178-L178)

This architectural design provides a balance between performance, safety, and usability, making the cpumask library suitable for high-performance operating system kernels while maintaining Rust's safety guarantees.