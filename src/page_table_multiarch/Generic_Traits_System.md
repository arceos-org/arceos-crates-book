# Generic Traits System

> **Relevant source files**
> * [page_table_entry/src/lib.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/lib.rs)
> * [page_table_multiarch/src/lib.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs)

This document describes the core trait system that enables architecture abstraction in the page_table_multiarch library. The traits define interfaces for architecture-specific metadata, OS-level memory management, and page table entry manipulation, allowing a single `PageTable64` implementation to work across multiple processor architectures.

For information about how these traits are implemented for specific architectures, see the Architecture Support section [4](/arceos-org/page_table_multiarch/4-architecture-support). For details about the main page table implementation that uses these traits, see [PageTable64 Implementation](/arceos-org/page_table_multiarch/3.1-pagetable64-implementation).

## Core Trait Architecture

The generic traits system consists of three primary traits that decouple architecture-specific behavior from the common page table logic:

```mermaid
flowchart TD
subgraph subGraph3["Architecture Implementations"]
    X86_MD["X64PagingMetaData"]
    ARM_MD["A64PagingMetaData"]
    RV_MD["Sv39MetaData/Sv48MetaData"]
    LA_MD["LA64MetaData"]
    X86_PTE["X64PTE"]
    ARM_PTE["A64PTE"]
    RV_PTE["Rv64PTE"]
    LA_PTE["LA64PTE"]
end
subgraph subGraph2["Page Table Implementation"]
    PT64["PageTable64<M,PTE,H>Generic page table"]
end
subgraph subGraph1["Supporting Types"]
    MF["MappingFlagsGeneric permission flags"]
    PS["PageSize4K, 2M, 1G page sizes"]
    TLB["TlbFlush/TlbFlushAllTLB invalidation"]
end
subgraph subGraph0["Generic Traits Layer"]
    PMD["PagingMetaDataArchitecture constants& address validation"]
    PH["PagingHandlerOS memory management& virtual mapping"]
    GPTE["GenericPTEPage table entrymanipulation"]
end

GPTE --> ARM_PTE
GPTE --> LA_PTE
GPTE --> RV_PTE
GPTE --> X86_PTE
PMD --> ARM_MD
PMD --> LA_MD
PMD --> RV_MD
PMD --> X86_MD
PT64 --> GPTE
PT64 --> MF
PT64 --> PH
PT64 --> PMD
PT64 --> PS
PT64 --> TLB
```

*Sources: [page_table_multiarch/src/lib.rs(L42 - L92)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L42-L92) [page_table_entry/src/lib.rs(L41 - L68)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/lib.rs#L41-L68)*

## PagingMetaData Trait

The `PagingMetaData` trait defines architecture-specific constants and validation logic for page table implementations:

|Method/Constant|Purpose|Type|
| --- | --- | --- |
|LEVELS|Number of page table levels|const usize|
|PA_MAX_BITS|Maximum physical address bits|const usize|
|VA_MAX_BITS|Maximum virtual address bits|const usize|
|VirtAddr|Virtual address type for this architecture|Associated type|
|paddr_is_valid()|Validates physical addresses|fn(usize) -> bool|
|vaddr_is_valid()|Validates virtual addresses|fn(usize) -> bool|
|flush_tlb()|Flushes Translation Lookaside Buffer|fn(Option<Self::VirtAddr>)|

### Address Validation Logic

The trait provides default implementations for address validation that work for most architectures:

```mermaid
flowchart TD
PADDR_CHECK["paddr_is_valid(paddr)"]
PADDR_RESULT["paddr <= PA_MAX_ADDR"]
VADDR_CHECK["vaddr_is_valid(vaddr)"]
VADDR_MASK["top_mask = usize::MAX << (VA_MAX_BITS - 1)"]
VADDR_SIGN["(vaddr & top_mask) == 0 ||(vaddr & top_mask) == top_mask"]

PADDR_CHECK --> PADDR_RESULT
VADDR_CHECK --> VADDR_MASK
VADDR_MASK --> VADDR_SIGN
```

*Sources: [page_table_multiarch/src/lib.rs(L61 - L72)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L61-L72)*

## PagingHandler Trait

The `PagingHandler` trait abstracts OS-level memory management operations required by the page table implementation:

|Method|Purpose|Signature|
| --- | --- | --- |
|alloc_frame()|Allocate a 4K physical frame|fn() -> Option<PhysAddr>|
|dealloc_frame()|Free a physical frame|fn(PhysAddr)|
|phys_to_virt()|Get virtual address for physical memory access|fn(PhysAddr) -> VirtAddr|

This trait enables the page table implementation to work with different memory management systems by requiring the OS to provide these three fundamental operations.

*Sources: [page_table_multiarch/src/lib.rs(L83 - L92)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L83-L92)*

## GenericPTE Trait

The `GenericPTE` trait provides a unified interface for manipulating page table entries across different architectures:

```mermaid
flowchart TD
subgraph subGraph4["Query Methods"]
    IS_UNUSED["is_unused() -> bool"]
    IS_PRESENT["is_present() -> bool"]
    IS_HUGE["is_huge() -> bool"]
end
subgraph subGraph3["Modification Methods"]
    SET_PADDR["set_paddr(paddr)"]
    SET_FLAGS["set_flags(flags, is_huge)"]
    CLEAR["clear()"]
end
subgraph subGraph2["Access Methods"]
    GET_PADDR["paddr() -> PhysAddr"]
    GET_FLAGS["flags() -> MappingFlags"]
    GET_BITS["bits() -> usize"]
end
subgraph subGraph1["Creation Methods"]
    NEW_PAGE["new_page(paddr, flags, is_huge)"]
    NEW_TABLE["new_table(paddr)"]
end
subgraph subGraph0["GenericPTE Interface"]
    CREATE["Creation Methods"]
    ACCESS["Access Methods"]
    MODIFY["Modification Methods"]
    QUERY["Query Methods"]
end

ACCESS --> GET_BITS
ACCESS --> GET_FLAGS
ACCESS --> GET_PADDR
CREATE --> NEW_PAGE
CREATE --> NEW_TABLE
MODIFY --> CLEAR
MODIFY --> SET_FLAGS
MODIFY --> SET_PADDR
QUERY --> IS_HUGE
QUERY --> IS_PRESENT
QUERY --> IS_UNUSED
```

*Sources: [page_table_entry/src/lib.rs(L41 - L68)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/lib.rs#L41-L68)*

## MappingFlags System

The `MappingFlags` bitflags provide a generic representation of memory permissions and attributes:

|Flag|Value|Purpose|
| --- | --- | --- |
|READ|1 << 0|Memory is readable|
|WRITE|1 << 1|Memory is writable|
|EXECUTE|1 << 2|Memory is executable|
|USER|1 << 3|Memory is user-accessible|
|DEVICE|1 << 4|Memory is device memory|
|UNCACHED|1 << 5|Memory is uncached|

Architecture-specific implementations convert between these generic flags and their hardware-specific representations.

*Sources: [page_table_entry/src/lib.rs(L12 - L30)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/lib.rs#L12-L30)*

## Page Size Support

The `PageSize` enum defines supported page sizes across architectures:

```mermaid
flowchart TD
subgraph subGraph0["Helper Methods"]
    IS_HUGE["is_huge() -> bool"]
    IS_ALIGNED["is_aligned(addr) -> bool"]
    ALIGN_OFFSET["align_offset(addr) -> usize"]
end
PS["PageSize enum"]
SIZE_4K["Size4K = 0x1000(4 KiB)"]
SIZE_2M["Size2M = 0x20_0000(2 MiB)"]
SIZE_1G["Size1G = 0x4000_0000(1 GiB)"]

PS --> SIZE_1G
PS --> SIZE_2M
PS --> SIZE_4K
SIZE_1G --> IS_HUGE
SIZE_2M --> IS_HUGE
```

*Sources: [page_table_multiarch/src/lib.rs(L95 - L128)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L95-L128)*

## TLB Management Types

The system provides type-safe TLB invalidation through must-use wrapper types:

```mermaid
flowchart TD
subgraph Actions["Actions"]
    FLUSH["flush() - Execute flush"]
    IGNORE["ignore() - Skip flush"]
    FLUSH_ALL["flush_all() - Execute full flush"]
end
subgraph subGraph0["TLB Flush Types"]
    TLB_FLUSH["TlbFlush<M>Single address flush"]
    TLB_FLUSH_ALL["TlbFlushAll<M>Complete TLB flush"]
end

TLB_FLUSH --> FLUSH
TLB_FLUSH --> IGNORE
TLB_FLUSH_ALL --> FLUSH_ALL
TLB_FLUSH_ALL --> IGNORE
```

These types ensure that TLB flushes are not accidentally forgotten after page table modifications by using the `#[must_use]` attribute.

*Sources: [page_table_multiarch/src/lib.rs(L130 - L172)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L130-L172)*

## Trait Integration Pattern

The three core traits work together to parameterize the `PageTable64` implementation:

```

```

This design allows compile-time specialization while maintaining a common interface, enabling zero-cost abstractions across different processor architectures.

*Sources: [page_table_multiarch/src/lib.rs(L42 - L92)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L42-L92) [page_table_entry/src/lib.rs(L41 - L68)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/lib.rs#L41-L68)*