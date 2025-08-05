# System Architecture

> **Relevant source files**
> * [page_table_entry/src/lib.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/lib.rs)
> * [page_table_multiarch/README.md](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/README.md)
> * [page_table_multiarch/src/lib.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs)

This document explains the overall design philosophy, abstraction layers, and architecture independence mechanisms of the `page_table_multiarch` library. The purpose is to provide a comprehensive understanding of how the system achieves unified page table management across multiple processor architectures through a layered abstraction approach.

For detailed information about specific processor architectures, see [Architecture Support](/arceos-org/page_table_multiarch/4-architecture-support). For implementation details of the core abstractions, see [Core Concepts](/arceos-org/page_table_multiarch/3-core-concepts).

## Design Philosophy

The `page_table_multiarch` library implements a generic, unified, architecture-independent approach to page table management. The system separates architecture-specific concerns from generic page table operations through a trait-based abstraction layer that allows the same high-level API to work across x86_64, AArch64, RISC-V, and LoongArch64 platforms.

### Core Abstraction Model

```mermaid
flowchart TD
subgraph subGraph4["Hardware Layer"]
    X86HW["x86_64 MMU"]
    ARMHW["AArch64 MMU"]
    RVHW["RISC-V MMU"]
    LAHW["LoongArch64 MMU"]
end
subgraph subGraph3["Architecture Implementation Layer"]
    X86Meta["X64PagingMetaData"]
    ARMeta["A64PagingMetaData"]
    RVMeta["Sv39/Sv48MetaData"]
    LAMeta["LA64MetaData"]
    X86PTE["X64PTE"]
    ARMPTE["A64PTE"]
    RVPTE["Rv64PTE"]
    LAPTE["LA64PTE"]
end
subgraph subGraph2["Trait Abstraction Layer"]
    PMD["PagingMetaData trait"]
    GPTE["GenericPTE trait"]
    PH["PagingHandler trait"]
end
subgraph subGraph1["Generic Interface Layer"]
    PT64["PageTable64<M,PTE,H>"]
    API["Unified API Methods"]
end
subgraph subGraph0["Application Layer"]
    App["OS/Hypervisor Code"]
end

API --> GPTE
API --> PH
API --> PMD
ARMPTE --> ARMHW
ARMeta --> ARMHW
App --> PT64
GPTE --> ARMPTE
GPTE --> LAPTE
GPTE --> RVPTE
GPTE --> X86PTE
LAMeta --> LAHW
LAPTE --> LAHW
PMD --> ARMeta
PMD --> LAMeta
PMD --> RVMeta
PMD --> X86Meta
PT64 --> API
RVMeta --> RVHW
RVPTE --> RVHW
X86Meta --> X86HW
X86PTE --> X86HW
```

Sources: [page_table_multiarch/src/lib.rs(L9 - L19)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L9-L19) [page_table_multiarch/README.md(L9 - L20)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/README.md#L9-L20)

## Workspace Architecture

The system is organized as a two-crate Cargo workspace that separates high-level page table management from low-level page table entry definitions:

### Crate Dependency Structure

```mermaid
flowchart TD
subgraph subGraph3["page_table_multiarch Workspace"]
    PTM["page_table_multiarch crate"]
    PTELib["lib.rs"]
    PTEArch["arch/ modules"]
    PTMLib["lib.rs"]
    PTMBits["bits64.rs"]
    subgraph subGraph2["External Dependencies"]
        MemAddr["memory_addr"]
        Log["log"]
        Bitflags["bitflags"]
    end
    subgraph subGraph0["PTM Modules"]
        PTE["page_table_entry crate"]
        PTMArch["arch/ modules"]
        subgraph subGraph1["PTE Modules"]
            PTM["page_table_multiarch crate"]
            PTELib["lib.rs"]
            PTEArch["arch/ modules"]
            PTMLib["lib.rs"]
            PTMBits["bits64.rs"]
        end
    end
end

PTE --> Bitflags
PTE --> MemAddr
PTELib --> PTEArch
PTM --> Log
PTM --> MemAddr
PTM --> PTE
PTMLib --> PTMArch
PTMLib --> PTMBits
```

|Crate|Purpose|Key Exports|
| --- | --- | --- |
|page_table_multiarch|High-level page table abstractions|PageTable64,PagingMetaData,PagingHandler|
|page_table_entry|Low-level page table entry definitions|GenericPTE,MappingFlags|

Sources: [page_table_multiarch/src/lib.rs(L15 - L19)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L15-L19) [page_table_entry/src/lib.rs(L10)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/lib.rs#L10-L10)

## Core Trait System

The architecture independence is achieved through three primary traits that define contracts between generic and architecture-specific code:

### Trait Relationships and Responsibilities

```mermaid
flowchart TD
subgraph subGraph2["Trait Methods & Constants"]
    PMDMethods["LEVELS: usizePA_MAX_BITS: usizeVA_MAX_BITS: usizeVirtAddr: MemoryAddrflush_tlb()"]
    GPTEMethods["new_page()new_table()paddr()flags()is_present()is_huge()"]
    PHMethods["alloc_frame()dealloc_frame()phys_to_virt()"]
end
subgraph subGraph1["Core Traits"]
    PMDTrait["PagingMetaData"]
    GPTETrait["GenericPTE"]
    PHTrait["PagingHandler"]
end
subgraph subGraph0["Generic Types"]
    PT64Struct["PageTable64<M,PTE,H>"]
    TlbFlush["TlbFlush<M>"]
    TlbFlushAll["TlbFlushAll<M>"]
    MFlags["MappingFlags"]
    PSize["PageSize"]
end

GPTETrait --> GPTEMethods
GPTETrait --> MFlags
PHTrait --> PHMethods
PMDTrait --> PMDMethods
PT64Struct --> GPTETrait
PT64Struct --> PHTrait
PT64Struct --> PMDTrait
TlbFlush --> PMDTrait
TlbFlushAll --> PMDTrait
```

### Trait Responsibilities

|Trait|Responsibility|Key Types|
| --- | --- | --- |
|PagingMetaData|Architecture constants and TLB operations|LEVELS,PA_MAX_BITS,VA_MAX_BITS,VirtAddr|
|GenericPTE|Page table entry manipulation|Entry creation, flag handling, address extraction|
|PagingHandler|OS-dependent memory operations|Frame allocation, virtual-physical address translation|

Sources: [page_table_multiarch/src/lib.rs(L40 - L92)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L40-L92) [page_table_entry/src/lib.rs(L38 - L68)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/lib.rs#L38-L68)

## Architecture Independence Mechanisms

### Generic Parameter System

The `PageTable64<M, PTE, H>` struct uses three generic parameters to achieve architecture independence:

```
// From page_table_multiarch/src/lib.rs and bits64.rs
PageTable64<M: PagingMetaData, PTE: GenericPTE, H: PagingHandler>
```

* **`M: PagingMetaData`** - Provides architecture-specific constants and TLB operations
* **`PTE: GenericPTE`** - Handles architecture-specific page table entry formats
* **`H: PagingHandler`** - Abstracts OS-specific memory management operations

### Architecture-Specific Implementations

Each supported architecture provides concrete implementations of the core traits:

|Architecture|Metadata Type|PTE Type|Example Usage|
| --- | --- | --- | --- |
|x86_64|X64PagingMetaData|X64PTE|X64PageTable|
|AArch64|A64PagingMetaData|A64PTE|A64PageTable|
|RISC-V Sv39|Sv39MetaData|Rv64PTE|Sv39PageTable|
|RISC-V Sv48|Sv48MetaData|Rv64PTE|Sv48PageTable|
|LoongArch64|LA64MetaData|LA64PTE|LA64PageTable|

### Error Handling and Type Safety

The system defines a comprehensive error model through `PagingError` and `PagingResult` types:

```mermaid
flowchart TD
subgraph subGraph1["Error Variants"]
    NoMemory["NoMemory"]
    NotAligned["NotAligned"]
    NotMapped["NotMapped"]
    AlreadyMapped["AlreadyMapped"]
    MappedToHuge["MappedToHugePage"]
end
subgraph subGraph0["Error Types"]
    PError["PagingError"]
    PResult["PagingResult<T>"]
end

PError --> AlreadyMapped
PError --> MappedToHuge
PError --> NoMemory
PError --> NotAligned
PError --> NotMapped
PResult --> PError
```

Sources: [page_table_multiarch/src/lib.rs(L21 - L38)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L21-L38)

## TLB Management Architecture

The system implements a type-safe TLB (Translation Lookaside Buffer) management mechanism through specialized wrapper types:

### TLB Flush Types

```mermaid
flowchart TD
subgraph subGraph2["Architecture Implementation"]
    FlushTLB["M::flush_tlb(Option<VirtAddr>)"]
end
subgraph subGraph1["TLB Operations"]
    FlushSingle["flush() - Single address"]
    FlushAll["flush_all() - Entire TLB"]
    Ignore["ignore() - Skip flush"]
end
subgraph subGraph0["TLB Management Types"]
    TlbFlush["TlbFlush<M>"]
    TlbFlushAll["TlbFlushAll<M>"]
end

FlushAll --> FlushTLB
FlushSingle --> FlushTLB
TlbFlush --> FlushSingle
TlbFlush --> Ignore
TlbFlushAll --> FlushAll
TlbFlushAll --> Ignore
```

The `#[must_use]` attribute ensures that TLB flush operations are not accidentally ignored, promoting system correctness.

Sources: [page_table_multiarch/src/lib.rs(L130 - L172)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L130-L172)

## Page Size Support

The system supports multiple page sizes through the `PageSize` enumeration:

|Page Size|Value|Usage|
| --- | --- | --- |
|Size4K|4 KiB (0x1000)|Standard pages|
|Size2M|2 MiB (0x200000)|Huge pages|
|Size1G|1 GiB (0x40000000)|Huge pages|

The `PageSize::is_huge()` method distinguishes between standard and huge pages, enabling architecture-specific optimizations for large memory mappings.

Sources: [page_table_multiarch/src/lib.rs(L94 - L128)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/lib.rs#L94-L128)