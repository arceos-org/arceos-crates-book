# Address Ranges

> **Relevant source files**
> * [memory_addr/src/range.rs](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_addr/src/range.rs)

This document covers the address range functionality provided by the `memory_addr` crate, specifically the `AddrRange` generic type and its associated operations. Address ranges represent contiguous spans of memory addresses with type safety and comprehensive range manipulation operations.

For information about individual address types and arithmetic operations, see [Address Types and Operations](/arceos-org/axmm_crates/2.1-address-types-and-operations). For page iteration over ranges, see [Page Iteration](/arceos-org/axmm_crates/2.3-page-iteration).

## Address Range Structure

The core of the address range system is the `AddrRange<A>` generic struct, where `A` implements the `MemoryAddr` trait. This provides type-safe representation of memory address ranges with inclusive start bounds and exclusive end bounds.

### AddrRange Type Hierarchy

```mermaid
flowchart TD
subgraph subGraph2["Address Types"]
    VA["VirtAddr"]
    PA["PhysAddr"]
    UA["usize"]
end
subgraph subGraph1["Concrete Type Aliases"]
    VAR["VirtAddrRangeAddrRange<VirtAddr>"]
    PAR["PhysAddrRangeAddrRange<PhysAddr>"]
end
subgraph subGraph0["Generic Types"]
    AR["AddrRange<A>Generic range type"]
    MA["MemoryAddr traitConstraint for A"]
end

AR --> MA
AR --> UA
PA --> MA
PAR --> AR
PAR --> PA
UA --> MA
VA --> MA
VAR --> AR
VAR --> VA
```

Sources: [memory_addr/src/range.rs(L1 - L28)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_addr/src/range.rs#L1-L28) [memory_addr/src/range.rs(L365 - L368)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_addr/src/range.rs#L365-L368)

The `AddrRange<A>` struct contains two public fields:

* `start: A` - The lower bound of the range (inclusive)
* `end: A` - The upper bound of the range (exclusive)

Type aliases provide convenient names for common address range types:

* `VirtAddrRange` = `AddrRange<VirtAddr>`
* `PhysAddrRange` = `AddrRange<PhysAddr>`

## Range Construction Methods

Address ranges can be constructed through several methods, each with different safety and error-handling characteristics.

### Construction Method Categories

|Method|Safety|Overflow Handling|Use Case|
| --- | --- | --- | --- |
|new()|Panics on invalid range|Checked|Safe construction with panic|
|try_new()|ReturnsOption|Checked|Fallible construction|
|new_unchecked()|Unsafe|Unchecked|Performance-critical paths|
|from_start_size()|Panics on overflow|Checked|Size-based construction|
|try_from_start_size()|ReturnsOption|Checked|Fallible size-based construction|
|from_start_size_unchecked()|Unsafe|Unchecked|Performance-critical size-based|

Sources: [memory_addr/src/range.rs(L34 - L193)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_addr/src/range.rs#L34-L193)

### Range Construction Flow

```mermaid
flowchart TD
subgraph Results["Results"]
    SUCCESS["AddrRange<A>"]
    PANIC["Panic"]
    NONE["None"]
    ERROR["Error"]
end
subgraph Validation["Validation"]
    CHECK["start <= end?"]
    OVERFLOW["Addition overflow?"]
end
subgraph subGraph1["Construction Methods"]
    NEW["new()"]
    TRYNEW["try_new()"]
    UNSAFE["new_unchecked()"]
    FSS["from_start_size()"]
    TRYFSS["try_from_start_size()"]
    UNSAFEFSS["from_start_size_unchecked()"]
    TRYFROM["TryFrom<Range<T>>"]
end
subgraph subGraph0["Input Types"]
    SE["start, end addresses"]
    SS["start address, size"]
    RNG["Rust Range<T>"]
end
START["Range Construction Request"]

CHECK --> NONE
CHECK --> PANIC
CHECK --> SUCCESS
FSS --> OVERFLOW
NEW --> CHECK
OVERFLOW --> NONE
OVERFLOW --> PANIC
OVERFLOW --> SUCCESS
RNG --> TRYFROM
SE --> NEW
SE --> TRYNEW
SE --> UNSAFE
SS --> FSS
SS --> TRYFSS
SS --> UNSAFEFSS
START --> RNG
START --> SE
START --> SS
TRYFROM --> CHECK
TRYFSS --> OVERFLOW
TRYNEW --> CHECK
UNSAFE --> SUCCESS
UNSAFEFSS --> SUCCESS
```

Sources: [memory_addr/src/range.rs(L57 - L193)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_addr/src/range.rs#L57-L193) [memory_addr/src/range.rs(L307 - L317)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_addr/src/range.rs#L307-L317)

## Range Operations and Relationships

Address ranges support comprehensive operations for checking containment, overlap, and spatial relationships between ranges.

### Range Relationship Operations

```mermaid
flowchart TD
subgraph Results["Results"]
    BOOL["bool"]
    USIZE["usize"]
end
subgraph subGraph1["Query Operations"]
    CONTAINS["contains(addr: A)Point containment"]
    CONTAINSRANGE["contains_range(other)Range containment"]
    CONTAINEDIN["contained_in(other)Reverse containment"]
    OVERLAPS["overlaps(other)Range intersection"]
    ISEMPTY["is_empty()Zero-size check"]
    SIZE["size()Range size"]
end
subgraph subGraph0["Range A"]
    RA["AddrRange<A>"]
end

CONTAINEDIN --> BOOL
CONTAINS --> BOOL
CONTAINSRANGE --> BOOL
ISEMPTY --> BOOL
OVERLAPS --> BOOL
RA --> CONTAINEDIN
RA --> CONTAINS
RA --> CONTAINSRANGE
RA --> ISEMPTY
RA --> OVERLAPS
RA --> SIZE
SIZE --> USIZE
```

Sources: [memory_addr/src/range.rs(L228 - L302)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_addr/src/range.rs#L228-L302)

### Key Range Operations

The following operations are available on all `AddrRange<A>` instances:

* **Point Containment**: `contains(addr)` checks if a single address falls within the range
* **Range Containment**: `contains_range(other)` checks if another range is entirely within this range
* **Containment Check**: `contained_in(other)` checks if this range is entirely within another range
* **Overlap Detection**: `overlaps(other)` checks if two ranges have any intersection
* **Empty Range Check**: `is_empty()` returns true if start equals end
* **Size Calculation**: `size()` returns the number of bytes in the range

## Convenience Macros

The crate provides three macros for convenient address range creation with compile-time type inference and runtime validation.

### Macro Overview

|Macro|Target Type|Purpose|
| --- | --- | --- |
|addr_range!|AddrRange<A>(inferred)|Generic range creation|
|va_range!|VirtAddrRange|Virtual address range creation|
|pa_range!|PhysAddrRange|Physical address range creation|

Sources: [memory_addr/src/range.rs(L391 - L448)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_addr/src/range.rs#L391-L448)

### Macro Usage Pattern

```mermaid
flowchart TD
subgraph subGraph3["Output Types"]
    GENERICRANGE["AddrRange<A> (inferred)"]
    VIRTRANGE["VirtAddrRange"]
    PHYSRANGE["PhysAddrRange"]
end
subgraph subGraph2["Internal Processing"]
    TRYFROM["TryFrom<Range<T>>::try_from()"]
    EXPECT["expect() on Result"]
end
subgraph subGraph1["Macro Processing"]
    ADDRRANGE["addr_range!(0x1000..0x2000)"]
    VARANGE["va_range!(0x1000..0x2000)"]
    PARANGE["pa_range!(0x1000..0x2000)"]
end
subgraph subGraph0["Input Syntax"]
    RANGE["Rust range syntaxstart..end"]
end

ADDRRANGE --> TRYFROM
EXPECT --> GENERICRANGE
EXPECT --> PHYSRANGE
EXPECT --> VIRTRANGE
PARANGE --> TRYFROM
RANGE --> ADDRRANGE
RANGE --> PARANGE
RANGE --> VARANGE
TRYFROM --> EXPECT
VARANGE --> TRYFROM
```

Sources: [memory_addr/src/range.rs(L391 - L448)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_addr/src/range.rs#L391-L448)

## Usage Examples and Patterns

The test suite demonstrates common usage patterns for address ranges, showing both basic operations and complex relationship checking.

### Basic Range Operations

```javascript
// Creating ranges with different methods
let range = VirtAddrRange::new(0x1000.into(), 0x2000.into());
let size_range = VirtAddrRange::from_start_size(0x1000.into(), 0x1000);

// Using macros for convenience
let macro_range = va_range!(0x1000..0x2000);

// Checking basic properties
assert_eq!(range.size(), 0x1000);
assert!(!range.is_empty());
assert!(range.contains(0x1500.into()));
```

Sources: [memory_addr/src/range.rs(L464 - L511)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_addr/src/range.rs#L464-L511)

### Range Relationship Testing

The codebase extensively tests range relationships using various scenarios:

```
// Containment checking
assert!(range.contains_range(va_range!(0x1001..0x1fff)));
assert!(!range.contains_range(va_range!(0xfff..0x2001)));

// Overlap detection
assert!(range.overlaps(va_range!(0x1800..0x2001)));
assert!(!range.overlaps(va_range!(0x2000..0x2800)));

// Spatial relationships
assert!(range.contained_in(va_range!(0xfff..0x2001)));
```

Sources: [memory_addr/src/range.rs(L483 - L505)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_addr/src/range.rs#L483-L505)

### Error Handling Patterns

The range system provides both panicking and fallible construction methods:

```javascript
// Panicking construction (for known-valid ranges)
let valid_range = VirtAddrRange::new(start, end);

// Fallible construction (for potentially invalid input)
if let Some(range) = VirtAddrRange::try_new(start, end) {
    // Use the valid range
}

// Size-based construction with overflow handling
let safe_range = VirtAddrRange::try_from_start_size(start, size)?;
```

Sources: [memory_addr/src/range.rs(L58 - L167)&emsp;](https://github.com/arceos-org/axmm_crates/blob/87b8ebcd/memory_addr/src/range.rs#L58-L167)