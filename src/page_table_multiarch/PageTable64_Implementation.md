# PageTable64 Implementation

> **Relevant source files**
> * [page_table_multiarch/src/bits64.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/bits64.rs)
> * [page_table_multiarch/src/lib.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs)

This document covers the core `PageTable64` struct that provides the main page table management functionality for 64-bit platforms in the page_table_multiarch library. It explains how the generic implementation handles multi-level page tables, memory mapping operations, and TLB management across different architectures.

For information about the trait system that enables architecture abstraction, see [Generic Traits System](/arceos-org/page_table_multiarch/3.2-generic-traits-system). For details about architecture-specific implementations, see [Architecture Support](/arceos-org/page_table_multiarch/4-architecture-support).

## Core Structure and Generic Parameters

The `PageTable64` struct serves as the primary interface for page table operations. It uses three generic parameters to achieve architecture independence while maintaining type safety.

```mermaid
flowchart TD
subgraph subGraph2["Core Operations"]
    MAP["map()Single page mapping"]
    UNMAP["unmap()Remove mapping"]
    QUERY["query()Lookup mapping"]
    PROTECT["protect()Change flags"]
    REGION["map_region()Bulk operations"]
end
subgraph subGraph1["Generic Parameters"]
    M["M: PagingMetaDataArchitecture constantsAddress validationTLB flushing"]
    PTE["PTE: GenericPTEPage table entryFlag manipulationAddress extraction"]
    H["H: PagingHandlerMemory allocationPhysical-virtual mapping"]
end
subgraph subGraph0["PageTable64 Structure"]
    PT64["PageTable64<M, PTE, H>"]
    ROOT["root_paddr: PhysAddr"]
    PHANTOM["_phantom: PhantomData<(M, PTE, H)>"]
end

H --> MAP
M --> MAP
MAP --> PROTECT
MAP --> QUERY
MAP --> REGION
MAP --> UNMAP
PT64 --> H
PT64 --> M
PT64 --> PHANTOM
PT64 --> PTE
PT64 --> ROOT
PTE --> MAP
```

The struct maintains only the physical address of the root page table, with all other state managed through the generic trait system. This design enables the same implementation to work across x86_64, AArch64, RISC-V, and LoongArch64 architectures.

*Sources: [page_table_multiarch/src/bits64.rs(L24 - L31)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/bits64.rs#L24-L31) [page_table_multiarch/src/lib.rs(L40 - L92)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L40-L92)*

## Page Table Hierarchy and Indexing

`PageTable64` supports both 3-level and 4-level page table configurations through constant indexing functions. The implementation uses bit manipulation to extract page table indices from virtual addresses.

```mermaid
flowchart TD
subgraph subGraph2["Page Table Levels"]
    ROOT["Root Table(P4 or P3)"]
    L3["Level 3 Table512 entries"]
    L2["Level 2 Table512 entriesCan map 2M pages"]
    L1["Level 1 Table512 entriesMaps 4K pages"]
end
subgraph subGraph1["Index Functions"]
    P4FUNC["p4_index()(vaddr >> 39) & 511"]
    P3FUNC["p3_index()(vaddr >> 30) & 511"]
    P2FUNC["p2_index()(vaddr >> 21) & 511"]
    P1FUNC["p1_index()(vaddr >> 12) & 511"]
end
subgraph subGraph0["Virtual Address Breakdown"]
    VADDR["Virtual Address (64-bit)"]
    P4["P4 Indexbits[48:39](4-level only)"]
    P3["P3 Indexbits[38:30]"]
    P2["P2 Indexbits[29:21]"]
    P1["P1 Indexbits[20:12]"]
    OFFSET["Page Offsetbits[11:0]"]
end

P1 --> P1FUNC
P1FUNC --> L1
P2 --> P2FUNC
P2FUNC --> L2
P3 --> P3FUNC
P3FUNC --> L3
P4 --> P4FUNC
P4FUNC --> ROOT
VADDR --> OFFSET
VADDR --> P1
VADDR --> P2
VADDR --> P3
VADDR --> P4
```

The system handles both 3-level configurations (like RISC-V Sv39) and 4-level configurations (like x86_64, AArch64) by checking `M::LEVELS` at compile time and selecting the appropriate starting table level.

*Sources: [page_table_multiarch/src/bits64.rs(L6 - L22)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/bits64.rs#L6-L22) [page_table_multiarch/src/bits64.rs(L401 - L426)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/bits64.rs#L401-L426)*

## Memory Mapping Operations

The core mapping functionality provides both single-page operations and bulk region operations. The implementation automatically handles huge page detection and creation of intermediate page tables.

```mermaid
sequenceDiagram
    participant User as User
    participant PageTable64 as PageTable64
    participant get_entry_mut_or_create as get_entry_mut_or_create
    participant GenericPTE as GenericPTE
    participant PagingHandler as PagingHandler

    Note over User,PagingHandler: Single Page Mapping Flow
    User ->> PageTable64: "map(vaddr, paddr, size, flags)"
    PageTable64 ->> get_entry_mut_or_create: "get_entry_mut_or_create(vaddr, size)"
    alt "Entry doesn't exist"
        get_entry_mut_or_create ->> PagingHandler: "alloc_frame()"
        PagingHandler -->> get_entry_mut_or_create: "PhysAddr"
        get_entry_mut_or_create ->> GenericPTE: "new_table(paddr)"
        get_entry_mut_or_create -->> PageTable64: "&mut PTE"
    else "Entry exists"
        get_entry_mut_or_create -->> PageTable64: "&mut PTE"
    end
    alt "Entry is unused"
        PageTable64 ->> GenericPTE: "new_page(paddr, flags, is_huge)"
        PageTable64 -->> User: "TlbFlush"
    else "Entry already mapped"
        PageTable64 -->> User: "Err(AlreadyMapped)"
    end
```

The mapping process supports three page sizes through the `PageSize` enum:

* `Size4K` (0x1000): Standard 4KB pages mapped at level 1
* `Size2M` (0x200000): 2MB huge pages mapped at level 2
* `Size1G` (0x40000000): 1GB huge pages mapped at level 3

*Sources: [page_table_multiarch/src/bits64.rs(L59 - L72)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/bits64.rs#L59-L72) [page_table_multiarch/src/bits64.rs(L455 - L484)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/bits64.rs#L455-L484) [page_table_multiarch/src/lib.rs(L94 - L128)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L94-L128)*

## Region Operations and Huge Page Optimization

The `map_region()` method provides optimized bulk mapping with automatic huge page selection when `allow_huge` is enabled. The implementation analyzes address alignment and region size to choose the largest possible page size.

|Page Size|Alignment Required|Minimum Size|Use Case|
| --- | --- | --- | --- |
|4K|4KB|4KB|General purpose mapping|
|2M|2MB|2MB|Large allocations, stack regions|
|1G|1GB|1GB|Kernel text, large data structures|

```mermaid
flowchart TD
START["map_region(vaddr, size, allow_huge)"]
CHECK_ALIGN["Is vaddr and paddr1G aligned?size >= 1G?"]
USE_1G["Use PageSize::Size1G"]
CHECK_2M["Is vaddr and paddr2M aligned?size >= 2M?"]
USE_2M["Use PageSize::Size2M"]
USE_4K["Use PageSize::Size4K"]
MAP["map(vaddr, paddr, page_size, flags)"]
UPDATE["vaddr += page_sizesize -= page_size"]
DONE["size == 0?"]

CHECK_2M --> USE_2M
CHECK_2M --> USE_4K
CHECK_ALIGN --> CHECK_2M
CHECK_ALIGN --> USE_1G
DONE --> CHECK_ALIGN
DONE --> START
MAP --> UPDATE
START --> CHECK_ALIGN
UPDATE --> DONE
USE_1G --> MAP
USE_2M --> MAP
USE_4K --> MAP
```

The algorithm prioritizes larger page sizes when possible, reducing TLB pressure and improving performance for large memory regions.

*Sources: [page_table_multiarch/src/bits64.rs(L157 - L217)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/bits64.rs#L157-L217) [page_table_multiarch/src/bits64.rs(L181 - L197)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/bits64.rs#L181-L197)*

## TLB Management and Flushing

The system provides fine-grained TLB management through the `TlbFlush` and `TlbFlushAll` types. These types enforce explicit handling of TLB invalidation to prevent stale translations.

```mermaid
flowchart TD
subgraph subGraph2["Flush Actions"]
    FLUSH["flush()Immediate TLB invalidation"]
    IGNORE["ignore()Defer to batch flush"]
end
subgraph subGraph1["Operations Returning Flushes"]
    MAP["map()"]
    UNMAP["unmap()"]
    PROTECT["protect()"]
    REMAP["remap()"]
    MAP_REGION["map_region()"]
    UNMAP_REGION["unmap_region()"]
end
subgraph subGraph0["TLB Flush Types"]
    SINGLE["TlbFlush<M>Single page flushContains VirtAddr"]
    ALL["TlbFlushAll<M>Complete TLB flushNo address needed"]
end

ALL --> FLUSH
ALL --> IGNORE
MAP --> SINGLE
MAP_REGION --> ALL
PROTECT --> SINGLE
REMAP --> SINGLE
SINGLE --> FLUSH
SINGLE --> IGNORE
UNMAP --> SINGLE
UNMAP_REGION --> ALL
```

The `#[must_use]` attribute on flush types ensures that TLB management is never accidentally ignored, preventing subtle bugs from stale page table entries.

*Sources: [page_table_multiarch/src/lib.rs(L130 - L172)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L130-L172) [page_table_multiarch/src/bits64.rs(L65 - L91)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/bits64.rs#L65-L91)*

## Memory Management and Cleanup

`PageTable64` implements automatic memory management through the `Drop` trait, ensuring all allocated page tables are freed when the structure is destroyed. The cleanup process uses recursive walking to identify and deallocate intermediate tables.

```mermaid
flowchart TD
subgraph subGraph1["Walk Process"]
    VISIT["Visit each table entry"]
    RECURSE["Recurse to next level"]
    CLEANUP["Post-order cleanup"]
end
subgraph subGraph0["Drop Implementation"]
    DROP["Drop::drop()"]
    WALK["walk() with post_func"]
    CHECK["level < LEVELS-1&&entry.is_present()&&!entry.is_huge()"]
    DEALLOC["H::dealloc_frame(entry.paddr())"]
    DEALLOC_ROOT["H::dealloc_frame(root_paddr)"]
end

CHECK --> CLEANUP
CHECK --> DEALLOC
CLEANUP --> CHECK
DEALLOC --> CLEANUP
DROP --> WALK
RECURSE --> CLEANUP
VISIT --> RECURSE
WALK --> DEALLOC_ROOT
WALK --> VISIT
```

The implementation carefully avoids freeing leaf-level entries, which point to actual data pages rather than page table structures that were allocated by the page table system itself.

*Sources: [page_table_multiarch/src/bits64.rs(L525 - L539)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/bits64.rs#L525-L539) [page_table_multiarch/src/bits64.rs(L313 - L325)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/bits64.rs#L313-L325)*

## Private Implementation Details

The `PageTable64` implementation uses several private helper methods to manage the complexity of multi-level page table traversal and memory allocation.

|Method|Purpose|Returns|
| --- | --- | --- |
|alloc_table()|Allocate and zero new page table|PagingResult<PhysAddr>|
|table_of()|Get immutable slice to table entries|&[PTE]|
|table_of_mut()|Get mutable slice to table entries|&mut [PTE]|
|next_table()|Follow entry to next level table|PagingResult<&[PTE]>|
|next_table_mut_or_create()|Get next level, creating if needed|PagingResult<&mut [PTE]>|
|get_entry()|Find entry for virtual address|PagingResult<(&PTE, PageSize)>|
|get_entry_mut()|Find mutable entry for virtual address|PagingResult<(&mut PTE, PageSize)>|

The helper methods handle error conditions like unmapped intermediate tables (`NotMapped`) and attempts to traverse through huge pages (`MappedToHugePage`), providing clear error semantics for page table operations.

*Sources: [page_table_multiarch/src/bits64.rs(L350 - L523)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/bits64.rs#L350-L523) [page_table_multiarch/src/lib.rs(L21 - L38)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L21-L38)*