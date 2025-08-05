# Query and Inspection Operations

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/cpumask/blob/a7cfa639/README.md)
> * [src/lib.rs](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs)

This section documents the read-only operations available on `CpuMask<SIZE>` instances for examining their state, testing individual bits, and finding set or unset CPU positions. These operations do not modify the cpumask and provide various ways to inspect its contents.

For information about creating new cpumask instances, see [Construction and Conversion Methods](/arceos-org/cpumask/2.1-construction-and-conversion-methods). For operations that modify cpumask state, see [Modification and Iteration](/arceos-org/cpumask/2.3-modification-and-iteration).

## Overview of Query Operations

The `CpuMask` struct provides comprehensive query capabilities organized into several functional categories. These operations leverage the underlying `bitmaps::Bitmap` functionality while providing a CPU-specific interface.

```mermaid
flowchart TD
subgraph subGraph1["Return Types"]
    U["bool"]
    V["bool / usize"]
    W["Option"]
    X["Option"]
    Y["Store / &Store / &[u8]"]
end
subgraph subGraph0["CpuMask Query API"]
    A["get(index)"]
    B["Basic Bit Testing"]
    C["len()"]
    D["Size and State Queries"]
    E["is_empty()"]
    F["is_full()"]
    G["first_index()"]
    H["True Bit Location"]
    I["last_index()"]
    J["next_index(index)"]
    K["prev_index(index)"]
    L["first_false_index()"]
    M["False Bit Location"]
    N["last_false_index()"]
    O["next_false_index(index)"]
    P["prev_false_index(index)"]
    Q["as_value()"]
    R["Value Access"]
    S["into_value()"]
    T["as_bytes()"]
end

A --> B
B --> U
C --> D
D --> V
E --> D
F --> D
G --> H
H --> W
I --> H
J --> H
K --> H
L --> M
M --> X
N --> M
O --> M
P --> M
Q --> R
R --> Y
S --> R
T --> R
```

Sources: [src/lib.rs(L148 - L234)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L148-L234)

## Basic Bit Testing

### get Method

The `get` method tests whether a specific CPU bit is set in the mask.

|Method|Signature|Description|Performance|
| --- | --- | --- | --- |
|get|get(self, index: usize) -> bool|Returnstrueif the bit atindexis set|O(1)|

The method performs bounds checking in debug builds and delegates to the underlying `BitsImpl<SIZE>::Store::get` implementation.

```mermaid
flowchart TD
A["CpuMask::get(index)"]
B["debug_assert!(index < SIZE)"]
C["self.into_value()"]
D["BitsImpl::Store::get(&value, index)"]
E["bool result"]

A --> B
B --> C
C --> D
D --> E
```

Sources: [src/lib.rs(L167 - L171)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L167-L171)

## Size and State Queries

These methods provide high-level information about the cpumask's state without requiring iteration through individual bits.

|Method|Signature|Description|Performance|
| --- | --- | --- | --- |
|len|len(self) -> usize|Count oftruebits in the mask|O(n) for most storage types|
|is_empty|is_empty(self) -> bool|Tests if no bits are set|O(log n)|
|is_full|is_full(self) -> bool|Tests if all valid bits are set|O(log n)|

### Implementation Details

The `is_empty` and `is_full` methods are implemented in terms of the index-finding operations:

```mermaid
flowchart TD
A["is_empty()"]
B["first_index().is_none()"]
C["is_full()"]
D["first_false_index().is_none()"]
E["Check if any bit is true"]
F["Check if any bit is false"]
G["len()"]
H["self.value.len()"]
I["Bitmap::len()"]

A --> B
B --> E
C --> D
D --> F
G --> H
H --> I
```

Sources: [src/lib.rs(L149 - L164)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L149-L164)

## Index Finding Operations

The cpumask provides efficient methods for locating set and unset bits, supporting both forward and backward traversal patterns commonly used in CPU scheduling algorithms.

### True Bit Location Methods

|Method|Signature|Description|
| --- | --- | --- |
|first_index|first_index(self) -> Option<usize>|Find first set bit|
|last_index|last_index(self) -> Option<usize>|Find last set bit|
|next_index|next_index(self, index: usize) -> Option<usize>|Find next set bit afterindex|
|prev_index|prev_index(self, index: usize) -> Option<usize>|Find previous set bit beforeindex|

### False Bit Location Methods

|Method|Signature|Description|
| --- | --- | --- |
|first_false_index|first_false_index(self) -> Option<usize>|Find first unset bit|
|last_false_index|last_false_index(self) -> Option<usize>|Find last unset bit|
|next_false_index|next_false_index(self, index: usize) -> Option<usize>|Find next unset bit afterindex|
|prev_false_index|prev_false_index(self, index: usize) -> Option<usize>|Find previous unset bit beforeindex|

### Delegation to BitsImpl

All index-finding operations delegate to the corresponding methods on the underlying storage type, with false-bit methods using corrected implementations that respect the `SIZE` boundary:

```mermaid
flowchart TD
subgraph subGraph2["Storage Access"]
    Q["self.into_value()"]
    R["as Bits>::Store"]
end
subgraph subGraph1["False Bit Methods"]
    I["first_false_index()"]
    M["BitsImpl::corrected_first_false_index()"]
    J["last_false_index()"]
    N["BitsImpl::corrected_last_false_index()"]
    K["next_false_index(index)"]
    O["BitsImpl::corrected_next_false_index()"]
    L["prev_false_index(index)"]
    P["BitsImpl::Store::prev_false_index()"]
end
subgraph subGraph0["True Bit Methods"]
    A["first_index()"]
    E["BitsImpl::Store::first_index()"]
    B["last_index()"]
    F["BitsImpl::Store::last_index()"]
    C["next_index(index)"]
    G["BitsImpl::Store::next_index()"]
    D["prev_index(index)"]
    H["BitsImpl::Store::prev_index()"]
end

A --> E
B --> F
C --> G
D --> H
E --> Q
F --> Q
G --> Q
H --> Q
I --> M
J --> N
K --> O
L --> P
M --> Q
N --> Q
O --> Q
P --> Q
Q --> R
```

Sources: [src/lib.rs(L183 - L228)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L183-L228)

## Value Access Operations

These methods provide direct access to the underlying storage representation of the cpumask, enabling interoperability with external systems and efficient bulk operations.

|Method|Signature|Description|Usage|
| --- | --- | --- | --- |
|into_value|into_value(self) -> <BitsImpl<SIZE> as Bits>::Store|Consume and return backing store|Move semantics|
|as_value|as_value(&self) -> &<BitsImpl<SIZE> as Bits>::Store|Reference to backing store|Borrowed access|
|as_bytes|as_bytes(&self) -> &[u8]|View as byte slice|Serialization|

### Storage Type Mapping

The actual storage type returned by `into_value` and referenced by `as_value` depends on the `SIZE` parameter:

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
J["into_value() -> bool"]
K["into_value() -> u8"]
L["into_value() -> u16"]
M["into_value() -> u32"]
N["into_value() -> u64"]
O["into_value() -> u128"]
P["into_value() -> [u128; N]"]

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
```

Sources: [src/lib.rs(L131 - L146)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L131-L146)

## Performance Characteristics

The query operations exhibit different performance characteristics based on the underlying storage type and operation complexity:

|Operation Category|Time Complexity|Notes|
| --- | --- | --- |
|get(index)|O(1)|Direct bit access|
|len()|O(n)|Requires bit counting|
|is_empty(),is_full()|O(log n)|Uses optimized bit scanning|
|Index finding|O(log n) to O(n)|Depends on bit density and hardware support|
|Value access|O(1)|Direct storage access|

The actual performance depends heavily on the target architecture's bit manipulation instructions and the density of set bits in the mask.

Sources: [src/lib.rs(L148 - L234)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L148-L234)