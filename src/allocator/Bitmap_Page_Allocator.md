# Bitmap Page Allocator

> **Relevant source files**
> * [src/bitmap.rs](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs)

This document covers the `BitmapPageAllocator` implementation, which provides page-granularity memory allocation using a bitmap-based approach. The allocator manages memory in fixed-size pages and uses an external bitmap data structure to track allocation status.

For byte-granularity allocation algorithms, see [Buddy System Allocator](/arceos-org/allocator/3.2-buddy-system-allocator), [Slab Allocator](/arceos-org/allocator/3.3-slab-allocator), and [TLSF Allocator](/arceos-org/allocator/3.4-tlsf-allocator). For broader architectural concepts and trait definitions, see [Architecture and Design](/arceos-org/allocator/2-architecture-and-design).

## Overview and Purpose

The `BitmapPageAllocator` is a page-granularity allocator that internally uses a bitmap where each bit indicates whether a page has been allocated. It wraps the external `bitmap_allocator` crate and implements the `PageAllocator` and `BaseAllocator` traits defined in the core library.

### Core Allocator Structure

```mermaid
classDiagram
class BitmapPageAllocator {
    +PAGE_SIZE: usize
    -base: usize
    -total_pages: usize
    -used_pages: usize
    -inner: BitAllocUsed
    +new() BitmapPageAllocator
    +alloc_pages(num_pages, align_pow2) AllocResult~usize~
    +alloc_pages_at(base, num_pages, align_pow2) AllocResult~usize~
    +dealloc_pages(pos, num_pages)
    +total_pages() usize
    +used_pages() usize
    +available_pages() usize
}

class BaseAllocator {
    <<interface>>
    
    +init(start, size)
    +add_memory(start, size) AllocResult
}

class PageAllocator {
    <<interface>>
    +PAGE_SIZE: usize
    +alloc_pages(num_pages, align_pow2) AllocResult~usize~
    +dealloc_pages(pos, num_pages)
    +total_pages() usize
    +used_pages() usize
    +available_pages() usize
}

class BitAllocUsed {
    <<external>>
    
    +alloc() Option~usize~
    +alloc_contiguous(start, num, align_log2) Option~usize~
    +dealloc(idx) bool
    +dealloc_contiguous(start, num) bool
    +insert(range)
}

PageAllocator  --|>  BaseAllocator
BitmapPageAllocator  --|>  PageAllocator
BitmapPageAllocator  *--  BitAllocUsed
```

Sources: [src/bitmap.rs(L34 - L50)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L34-L50) [src/bitmap.rs(L53 - L75)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L53-L75) [src/bitmap.rs(L77 - L160)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L77-L160)

## Memory Size Configuration

The allocator uses feature flags to configure the maximum memory size it can manage. Different `BitAlloc` implementations from the external `bitmap_allocator` crate are selected based on compilation features.

### Feature-Based BitAlloc Selection

```mermaid
flowchart TD
subgraph subGraph2["Memory Capacity"]
    CAP_4G["4GB Memory(Testing)"]
    CAP_1T["1TB Memory"]
    CAP_64G["64GB Memory"]
    CAP_256M["256MB Memory(Default)"]
end
subgraph subGraph1["BitAlloc Types"]
    BITALLOC_1M["BitAlloc1M1M pages = 4GB max"]
    BITALLOC_256M["BitAlloc256M256M pages = 1TB max"]
    BITALLOC_16M["BitAlloc16M16M pages = 64GB max"]
    BITALLOC_64K["BitAlloc64K64K pages = 256MB max"]
end
subgraph subGraph0["Feature Flags"]
    TEST["cfg(test)"]
    FEAT_1T["page-alloc-1t"]
    FEAT_64G["page-alloc-64g"]
    FEAT_4G["page-alloc-4g"]
    DEFAULT["default/page-alloc-256m"]
end

BITALLOC_16M --> CAP_64G
BITALLOC_1M --> CAP_4G
BITALLOC_256M --> CAP_1T
BITALLOC_64K --> CAP_256M
DEFAULT --> BITALLOC_64K
FEAT_1T --> BITALLOC_256M
FEAT_4G --> BITALLOC_1M
FEAT_64G --> BITALLOC_16M
TEST --> BITALLOC_1M
```

Sources: [src/bitmap.rs(L9 - L26)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L9-L26)

The configuration table shows the relationship between features and memory limits:

|Feature Flag|BitAlloc Type|Max Pages|Max Memory (4KB pages)|
| --- | --- | --- | --- |
|test|BitAlloc1M|1,048,576|4GB|
|page-alloc-1t|BitAlloc256M|268,435,456|1TB|
|page-alloc-64g|BitAlloc16M|16,777,216|64GB|
|page-alloc-4g|BitAlloc1M|1,048,576|4GB|
|page-alloc-256m(default)|BitAlloc64K|65,536|256MB|

## Core Implementation Details

### Memory Layout and Base Address Calculation

The allocator manages memory regions by calculating a base address aligned to 1GB boundaries and tracking page indices relative to this base.

```mermaid
flowchart TD
subgraph subGraph2["Bitmap Initialization"]
    INSERT_RANGE["inner.insert(start_idx..start_idx + total_pages)"]
end
subgraph subGraph1["Base Address Calculation"]
    CALC_BASE["base = align_down(start, MAX_ALIGN_1GB)"]
    REL_START["relative_start = start - base"]
    START_IDX["start_idx = relative_start / PAGE_SIZE"]
end
subgraph subGraph0["Memory Region Setup"]
    INPUT_START["Input: start, size"]
    ALIGN_START["align_up(start, PAGE_SIZE)"]
    ALIGN_END["align_down(start + size, PAGE_SIZE)"]
    CALC_TOTAL["total_pages = (end - start) / PAGE_SIZE"]
end

ALIGN_END --> CALC_TOTAL
ALIGN_START --> CALC_BASE
ALIGN_START --> CALC_TOTAL
CALC_BASE --> REL_START
CALC_TOTAL --> INSERT_RANGE
INPUT_START --> ALIGN_END
INPUT_START --> ALIGN_START
REL_START --> START_IDX
START_IDX --> INSERT_RANGE
```

Sources: [src/bitmap.rs(L54 - L70)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L54-L70) [src/bitmap.rs(L7)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L7-L7)

### Page Allocation Process

The allocator provides two allocation methods: general allocation with alignment requirements and allocation at specific addresses.

```mermaid
sequenceDiagram
    participant Client as Client
    participant BitmapPageAllocator as BitmapPageAllocator
    participant BitAllocUsed as BitAllocUsed

    Note over Client,BitAllocUsed: General Page Allocation
    Client ->> BitmapPageAllocator: alloc_pages(num_pages, align_pow2)
    BitmapPageAllocator ->> BitmapPageAllocator: Validate alignment parameters
    BitmapPageAllocator ->> BitmapPageAllocator: Convert align_pow2 to page units
    alt num_pages == 1
        BitmapPageAllocator ->> BitAllocUsed: alloc()
    else num_pages > 1
        BitmapPageAllocator ->> BitAllocUsed: alloc_contiguous(None, num_pages, align_log2)
    end
    BitAllocUsed -->> BitmapPageAllocator: Option<page_idx>
    BitmapPageAllocator ->> BitmapPageAllocator: Convert to address: idx * PAGE_SIZE + base
    BitmapPageAllocator ->> BitmapPageAllocator: Update used_pages count
    BitmapPageAllocator -->> Client: AllocResult<usize>
    Note over Client,BitAllocUsed: Specific Address Allocation
    Client ->> BitmapPageAllocator: alloc_pages_at(base, num_pages, align_pow2)
    BitmapPageAllocator ->> BitmapPageAllocator: Validate alignment and base address
    BitmapPageAllocator ->> BitmapPageAllocator: Calculate target page index
    BitmapPageAllocator ->> BitAllocUsed: alloc_contiguous(Some(idx), num_pages, align_log2)
    BitAllocUsed -->> BitmapPageAllocator: Option<page_idx>
    BitmapPageAllocator -->> Client: AllocResult<usize>
```

Sources: [src/bitmap.rs(L80 - L100)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L80-L100) [src/bitmap.rs(L103 - L131)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L103-L131)

## Allocation Constraints and Validation

The allocator enforces several constraints to ensure correct operation:

### Alignment Validation Process

```mermaid
flowchart TD
subgraph Results["Results"]
    VALID["Valid Parameters"]
    INVALID["AllocError::InvalidParam"]
end
subgraph subGraph2["Parameter Validation"]
    CHECK_NUM_PAGES["num_pages > 0"]
    CHECK_PAGE_SIZE["PAGE_SIZE.is_power_of_two()"]
end
subgraph subGraph1["Address Validation (alloc_pages_at)"]
    CHECK_BASE_ALIGN["is_aligned(base, align_pow2)"]
end
subgraph subGraph0["Alignment Checks"]
    CHECK_MAX["align_pow2 <= MAX_ALIGN_1GB"]
    CHECK_PAGE_ALIGN["is_aligned(align_pow2, PAGE_SIZE)"]
    CHECK_POW2["(align_pow2 / PAGE_SIZE).is_power_of_two()"]
end

CHECK_BASE_ALIGN --> INVALID
CHECK_BASE_ALIGN --> VALID
CHECK_MAX --> INVALID
CHECK_MAX --> VALID
CHECK_NUM_PAGES --> INVALID
CHECK_NUM_PAGES --> VALID
CHECK_PAGE_ALIGN --> INVALID
CHECK_PAGE_ALIGN --> VALID
CHECK_PAGE_SIZE --> VALID
CHECK_POW2 --> INVALID
CHECK_POW2 --> VALID
```

Sources: [src/bitmap.rs(L82 - L88)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L82-L88) [src/bitmap.rs(L111 - L121)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L111-L121) [src/bitmap.rs(L55)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L55-L55)

Key constraints include:

* `PAGE_SIZE` must be a power of two
* Maximum alignment is 1GB (`MAX_ALIGN_1GB = 0x4000_0000`)
* Alignment must be a multiple of `PAGE_SIZE`
* Alignment (in page units) must be a power of two
* For `alloc_pages_at`, the base address must be aligned to the requested alignment

## Memory Management Operations

### Deallocation Process

```mermaid
flowchart TD
subgraph subGraph1["Deallocation Flow"]
    INPUT["dealloc_pages(pos, num_pages)"]
    ASSERT_ALIGN["assert: pos aligned to PAGE_SIZE"]
    CALC_IDX["idx = (pos - base) / PAGE_SIZE"]
    UPDATE_COUNT["used_pages -= num_pages"]
    subgraph subGraph0["Bitmap Update"]
        SINGLE["num_pages == 1:inner.dealloc(idx)"]
        MULTI["num_pages > 1:inner.dealloc_contiguous(idx, num_pages)"]
    end
end

ASSERT_ALIGN --> CALC_IDX
CALC_IDX --> MULTI
CALC_IDX --> SINGLE
INPUT --> ASSERT_ALIGN
MULTI --> UPDATE_COUNT
SINGLE --> UPDATE_COUNT
```

Sources: [src/bitmap.rs(L133 - L147)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L133-L147)

### Statistics Tracking

The allocator maintains three key statistics accessible through the `PageAllocator` interface:

|Method|Description|Implementation|
| --- | --- | --- |
|total_pages()|Total pages managed|Returnsself.total_pages|
|used_pages()|Currently allocated pages|Returnsself.used_pages|
|available_pages()|Free pages remaining|Returnstotal_pages - used_pages|

Sources: [src/bitmap.rs(L149 - L159)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L149-L159)

## Usage Patterns and Testing

The test suite demonstrates typical usage patterns and validates the allocator's behavior across different scenarios.

### Single Page Allocation Example

```mermaid
sequenceDiagram
    participant TestCode as Test Code
    participant BitmapPageAllocator as BitmapPageAllocator

    TestCode ->> BitmapPageAllocator: new()
    TestCode ->> BitmapPageAllocator: init(PAGE_SIZE, PAGE_SIZE)
    Note over BitmapPageAllocator: total_pages=1, used_pages=0
    TestCode ->> BitmapPageAllocator: alloc_pages(1, PAGE_SIZE)
    BitmapPageAllocator -->> TestCode: addr = 0x1000
    Note over BitmapPageAllocator: used_pages=1, available_pages=0
    TestCode ->> BitmapPageAllocator: dealloc_pages(0x1000, 1)
    Note over BitmapPageAllocator: used_pages=0, available_pages=1
    TestCode ->> BitmapPageAllocator: alloc_pages(1, PAGE_SIZE)
    BitmapPageAllocator -->> TestCode: addr = 0x1000 (reused)
```

Sources: [src/bitmap.rs(L168 - L190)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L168-L190)

### Large Memory Region Testing

The test suite validates operation with large memory regions (2GB) and various allocation patterns:

* Sequential allocation/deallocation of 1, 10, 100, 1000 pages
* Alignment testing with powers of 2 from `PAGE_SIZE` to `MAX_ALIGN_1GB`
* Multiple concurrent allocations with different alignments
* Specific address allocation using `alloc_pages_at`

Sources: [src/bitmap.rs(L193 - L327)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/bitmap.rs#L193-L327)

The allocator demonstrates robust behavior across different memory sizes and allocation patterns while maintaining efficient bitmap-based tracking of page allocation status.