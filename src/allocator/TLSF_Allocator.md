# TLSF Allocator

> **Relevant source files**
> * [src/tlsf.rs](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/tlsf.rs)

This document covers the TLSF (Two-Level Segregated Fit) byte allocator implementation in the allocator crate. The `TlsfByteAllocator` provides real-time memory allocation with O(1) time complexity for both allocation and deallocation operations.

For information about other byte-level allocators, see [Buddy System Allocator](/arceos-org/allocator/3.2-buddy-system-allocator) and [Slab Allocator](/arceos-org/allocator/3.3-slab-allocator). For page-level allocation, see [Bitmap Page Allocator](/arceos-org/allocator/3.1-bitmap-page-allocator).

## TLSF Algorithm Overview

The Two-Level Segregated Fit (TLSF) algorithm is a real-time memory allocator that maintains free memory blocks in a two-level segregated list structure. It achieves deterministic O(1) allocation and deallocation performance by organizing free blocks into size classes using a first-level index (FLI) and second-level index (SLI).

### Algorithm Structure

```mermaid
flowchart TD
subgraph subGraph0["TLSF Algorithm Organization"]
    SLI1["Second Level Index 0"]
    SLI2["Second Level Index 1"]
    SLI3["Second Level Index N"]
    BLOCK1["Free Block 1"]
    BLOCK2["Free Block 2"]
    BLOCK3["Free Block 3"]
    BLOCKN["Free Block N"]
    subgraph subGraph1["Size Classification"]
        SMALL["Small blocks: 2^i to 2^(i+1)-1"]
        MEDIUM["Medium blocks: subdivided by SLI"]
        LARGE["Large blocks: power-of-2 ranges"]
        FLI["First Level Index (FLI)"]
    end
end

FLI --> SLI1
FLI --> SLI2
FLI --> SLI3
SLI1 --> BLOCK1
SLI1 --> BLOCK2
SLI2 --> BLOCK3
SLI3 --> BLOCKN
```

Sources: [src/tlsf.rs(L1 - L18)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/tlsf.rs#L1-L18)

## Implementation Structure

The `TlsfByteAllocator` is implemented as a wrapper around the `rlsf::Tlsf` external crate with specific configuration parameters optimized for the allocator crate's use cases.

### Core Components

```mermaid
flowchart TD
subgraph subGraph2["Trait Implementation"]
    BASE["BaseAllocator"]
    BYTE["ByteAllocator"]
    INIT["init()"]
    ADD["add_memory()"]
    ALLOC["alloc()"]
    DEALLOC["dealloc()"]
end
subgraph subGraph1["External Dependency"]
    RLSF["rlsf::Tlsf"]
    PARAMS["Generic Parameters"]
    FLLEN["FLLEN = 28"]
    SLLEN["SLLEN = 32"]
    MAXSIZE["Max Pool Size = 8GB"]
end
subgraph subGraph0["TlsfByteAllocator Structure"]
    WRAPPER["TlsfByteAllocator"]
    INNER["inner: Tlsf<'static, u32, u32, 28, 32>"]
    TOTAL["total_bytes: usize"]
    USED["used_bytes: usize"]
end

BASE --> ADD
BASE --> INIT
BYTE --> ALLOC
BYTE --> DEALLOC
INNER --> RLSF
PARAMS --> FLLEN
PARAMS --> MAXSIZE
PARAMS --> SLLEN
RLSF --> PARAMS
WRAPPER --> BASE
WRAPPER --> BYTE
WRAPPER --> INNER
WRAPPER --> TOTAL
WRAPPER --> USED
```

Sources: [src/tlsf.rs(L10 - L18)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/tlsf.rs#L10-L18) [src/tlsf.rs(L31 - L77)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/tlsf.rs#L31-L77)

### Configuration Parameters

The implementation uses fixed configuration parameters that define the allocator's capabilities:

|Parameter|Value|Purpose|
| --- | --- | --- |
|FLLEN|28|First-level index length, supporting up to 2^28 byte blocks|
|SLLEN|32|Second-level index length for fine-grained size classification|
|Max Pool Size|8GB|Theoretical maximum: 32 * 2^28 bytes|
|Generic Types|u32, u32|Index types for FLI and SLI bitmaps|

Sources: [src/tlsf.rs(L15)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/tlsf.rs#L15-L15)

## BaseAllocator Implementation

The `TlsfByteAllocator` implements the `BaseAllocator` trait to provide memory pool initialization and management capabilities.

### Memory Pool Management

```mermaid
flowchart TD
subgraph subGraph0["Memory Pool Operations"]
    INIT["init(start: usize, size: usize)"]
    RAW_PTR["Raw Memory Pointer"]
    ADD["add_memory(start: usize, size: usize)"]
    SLICE["core::slice::from_raw_parts_mut()"]
    NONNULL["NonNull<[u8]>"]
    INSERT["inner.insert_free_block_ptr()"]
    TOTAL_UPDATE["total_bytes += size"]
    ERROR_HANDLE["AllocError::InvalidParam"]
end

ADD --> RAW_PTR
INIT --> RAW_PTR
INSERT --> ERROR_HANDLE
INSERT --> TOTAL_UPDATE
NONNULL --> INSERT
RAW_PTR --> SLICE
SLICE --> NONNULL
```

The implementation converts raw memory addresses into safe slice references before inserting them into the TLSF data structure. Both `init()` and `add_memory()` follow the same pattern but with different error handling strategies.

Sources: [src/tlsf.rs(L31 - L51)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/tlsf.rs#L31-L51)

## ByteAllocator Implementation

The `ByteAllocator` trait implementation provides the core allocation and deallocation functionality with automatic memory usage tracking.

### Allocation Flow

```mermaid
flowchart TD
subgraph subGraph2["Memory Statistics"]
    TOTAL_BYTES["total_bytes()"]
    USED_BYTES["used_bytes()"]
    AVAIL_BYTES["available_bytes() = total - used"]
    DEALLOC_REQ["dealloc(pos: NonNull, layout: Layout)"]
    ALLOC_REQ["alloc(layout: Layout)"]
end
subgraph subGraph0["Allocation Process"]
    USED_BYTES["used_bytes()"]
    NO_MEMORY["Err(AllocError::NoMemory)"]
    RETURN_PTR["Ok(NonNull)"]
    subgraph subGraph1["Deallocation Process"]
        TOTAL_BYTES["total_bytes()"]
        DEALLOC_REQ["dealloc(pos: NonNull, layout: Layout)"]
        TLSF_DEALLOC["inner.deallocate(pos, layout.align())"]
        DECREASE_USED["used_bytes -= layout.size()"]
        ALLOC_REQ["alloc(layout: Layout)"]
        TLSF_ALLOC["inner.allocate(layout)"]
        PTR_CHECK["Result, ()>"]
        UPDATE_USED["used_bytes += layout.size()"]
    end
end

ALLOC_REQ --> TLSF_ALLOC
DEALLOC_REQ --> TLSF_DEALLOC
PTR_CHECK --> NO_MEMORY
PTR_CHECK --> UPDATE_USED
TLSF_ALLOC --> PTR_CHECK
TLSF_DEALLOC --> DECREASE_USED
UPDATE_USED --> RETURN_PTR
```

Sources: [src/tlsf.rs(L54 - L77)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/tlsf.rs#L54-L77)

### Memory Usage Tracking

The allocator maintains accurate memory usage statistics by tracking:

* **`total_bytes`**: Total memory pool size across all added memory regions
* **`used_bytes`**: Currently allocated memory, updated on each alloc/dealloc operation
* **`available_bytes`**: Computed as `total_bytes - used_bytes`

This tracking is performed at the wrapper level rather than delegating to the underlying `rlsf` implementation.

Sources: [src/tlsf.rs(L66 - L77)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/tlsf.rs#L66-L77)

## Performance Characteristics

The TLSF allocator provides several performance advantages:

### Time Complexity

|Operation|Complexity|Description|
| --- | --- | --- |
|Allocation|O(1)|Constant time regardless of pool size|
|Deallocation|O(1)|Constant time with immediate coalescing|
|Memory Addition|O(1)|Adding new memory pools|
|Statistics|O(1)|Memory usage queries|

### Memory Efficiency

The two-level segregated fit approach minimizes fragmentation by:

* Maintaining precise size classes for small allocations
* Using power-of-2 ranges for larger allocations
* Immediate coalescing of adjacent free blocks
* Good-fit allocation strategy to reduce waste

Sources: [src/tlsf.rs(L1 - L3)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/tlsf.rs#L1-L3)

## Configuration and Limitations

### Size Limitations

The current configuration imposes several limits:

* **Maximum single allocation**: Limited by `FLLEN = 28`, supporting up to 2^28 bytes (256MB)
* **Maximum total pool size**: 8GB theoretical limit (32 * 2^28)
* **Minimum allocation**: Determined by underlying `rlsf` implementation constraints

### Integration Requirements

The allocator requires:

* Feature gate `tlsf` to be enabled in `Cargo.toml`
* External dependency on `rlsf` crate version 0.2
* Unsafe operations for memory pool management
* `'static` lifetime for the internal TLSF structure

Sources: [src/tlsf.rs(L15)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/tlsf.rs#L15-L15) [src/tlsf.rs(L32 - L51)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/tlsf.rs#L32-L51)