# Overview

> **Relevant source files**
> * [CHANGELOG.md](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/CHANGELOG.md)
> * [Cargo.toml](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.toml)
> * [README.md](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/README.md)

## Purpose and Scope

The `page_table_multiarch` repository provides a generic, unified, architecture-independent page table management library for Rust systems programming. This library enables operating systems, hypervisors, and bare-metal applications to manage virtual memory translation across multiple hardware architectures through a single, consistent API.

The repository implements hardware abstraction for page table operations on x86_64, AArch64, RISC-V, and LoongArch64 architectures, providing both low-level page table entry manipulation and high-level page table management functionality. For detailed information about individual architecture implementations, see [Architecture Support](/arceos-org/page_table_multiarch/4-architecture-support). For development and contribution guidelines, see [Development Guide](/arceos-org/page_table_multiarch/5-development-guide).

Sources: [README.md(L1 - L16)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/README.md#L1-L16) [Cargo.toml(L17 - L18)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.toml#L17-L18)

## System Architecture

The library implements a layered architecture that separates generic page table operations from architecture-specific implementations through Rust's trait system.

### Core System Structure

```mermaid
flowchart TD
subgraph subGraph3["Architecture Implementations"]
    X86_IMPL["X64PageTableX64PTEX64PagingMetaData"]
    ARM_IMPL["A64PageTableA64PTEA64PagingMetaData"]
    RV_IMPL["Sv39PageTable, Sv48PageTableRv64PTESvPagingMetaData"]
    LA_IMPL["LA64PageTableLA64PTELA64PagingMetaData"]
end
subgraph subGraph2["Trait Abstraction Layer"]
    PMD["PagingMetaData trait"]
    GPTE["GenericPTE trait"]
    PH["PagingHandler trait"]
end
subgraph subGraph1["Generic API Layer"]
    PT64["PageTable64<M,PTE,H>"]
    API["map(), unmap(), protect()"]
    FLAGS["MappingFlags"]
end
subgraph subGraph0["Application Layer"]
    OS["Operating Systems"]
    HV["Hypervisors"]
    BM["Bare Metal Code"]
end

API --> GPTE
API --> PH
API --> PMD
BM --> PT64
GPTE --> ARM_IMPL
GPTE --> LA_IMPL
GPTE --> RV_IMPL
GPTE --> X86_IMPL
HV --> PT64
OS --> PT64
PH --> ARM_IMPL
PH --> LA_IMPL
PH --> RV_IMPL
PH --> X86_IMPL
PMD --> ARM_IMPL
PMD --> LA_IMPL
PMD --> RV_IMPL
PMD --> X86_IMPL
PT64 --> API
PT64 --> FLAGS
```

Sources: [Cargo.toml(L4 - L7)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.toml#L4-L7) [README.md(L5 - L10)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/README.md#L5-L10)

## Workspace Structure

The repository is organized as a Rust workspace containing two interdependent crates that together provide the complete page table management functionality.

### Crate Dependencies and Relationships

```mermaid
flowchart TD
subgraph subGraph5["page_table_multiarch Workspace"]
    WS["Cargo.tomlworkspace root"]
    PT64_TYPE["PageTable64<M,PTE,H>"]
    GPTE_TRAIT["GenericPTE trait"]
    ARCH_TABLES["X64PageTableA64PageTableSv39PageTableSv48PageTableLA64PageTable"]
    PTE_TYPES["X64PTEA64PTERv64PTELA64PTE"]
    FLAGS_TYPE["MappingFlags"]
    subgraph subGraph4["External Dependencies"]
        MEMADDR["memory_addr"]
        LOG["log"]
        BITFLAGS["bitflags"]
    end
    subgraph subGraph3["page_table_entry crate"]
        PTE["page_table_entry"]
        PTE_LIB["lib.rs"]
        PTE_ARCH["arch/ modules"]
        subgraph subGraph2["Low-level Types"]
            GPTE_TRAIT["GenericPTE trait"]
            PTE_TYPES["X64PTEA64PTERv64PTELA64PTE"]
            FLAGS_TYPE["MappingFlags"]
        end
    end
    subgraph subGraph1["page_table_multiarch crate"]
        PTM["page_table_multiarch"]
        PTM_LIB["lib.rs"]
        PTM_BITS["bits64.rs"]
        PTM_ARCH["arch/ modules"]
        subgraph subGraph0["High-level Types"]
            PT64_TYPE["PageTable64<M,PTE,H>"]
            ARCH_TABLES["X64PageTableA64PageTableSv39PageTableSv48PageTableLA64PageTable"]
        end
    end
end

ARCH_TABLES --> PTE_TYPES
GPTE_TRAIT --> FLAGS_TYPE
PT64_TYPE --> GPTE_TRAIT
PTE --> BITFLAGS
PTE --> MEMADDR
PTE_TYPES --> GPTE_TRAIT
PTM --> LOG
PTM --> MEMADDR
PTM --> PTE
WS --> PTE
WS --> PTM
```

The `page_table_multiarch` crate provides high-level page table management through the `PageTable64` generic struct and architecture-specific type aliases. The `page_table_entry` crate provides low-level page table entry definitions and the `GenericPTE` trait that enables architecture abstraction.

Sources: [Cargo.toml(L4 - L7)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.toml#L4-L7) [README.md(L12 - L15)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/README.md#L12-L15)

## Supported Architectures

The library supports four major processor architectures, each with specific paging characteristics and implementation details.

### Architecture Support Matrix

|Architecture|Page Table Levels|Virtual Address Width|Physical Address Width|Implementation Types|
| --- | --- | --- | --- | --- |
|x86_64|4 levels|48-bit|52-bit|X64PageTable,X64PTE|
|AArch64|4 levels|48-bit|48-bit|A64PageTable,A64PTE|
|RISC-V Sv39|3 levels|39-bit|56-bit|Sv39PageTable,Rv64PTE|
|RISC-V Sv48|4 levels|48-bit|56-bit|Sv48PageTable,Rv64PTE|
|LoongArch64|4 levels|48-bit|48-bit|LA64PageTable,LA64PTE|

Each architecture implementation provides the same generic interface through the trait system while handling architecture-specific page table formats, address translation mechanisms, and memory attribute encodings.

Sources: [README.md(L5 - L10)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/README.md#L5-L10) [CHANGELOG.md(L21)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/CHANGELOG.md#L21-L21)

## Key Abstractions

The library's architecture independence is achieved through three core traits that define the interface between generic and architecture-specific code.

### Trait System Overview

```mermaid
flowchart TD
subgraph subGraph4["Generic Implementation"]
    PT64_STRUCT["PageTable64<M: PagingMetaData,PTE: GenericPTE,H: PagingHandler>"]
end
subgraph subGraph3["PagingHandler Methods"]
    PH_METHODS["alloc_frame()dealloc_frame()phys_to_virt()"]
end
subgraph subGraph2["GenericPTE Methods"]
    GPTE_METHODS["new_page()new_table()paddr()flags()is_present()is_huge()empty()"]
end
subgraph subGraph1["PagingMetaData Methods"]
    PMD_METHODS["PAGE_SIZEVADDR_SIZEPADDR_SIZEENTRY_COUNTMAX_LEVEL"]
end
subgraph subGraph0["Core Trait Definitions"]
    PMD_TRAIT["PagingMetaData"]
    GPTE_TRAIT["GenericPTE"]
    PH_TRAIT["PagingHandler"]
end
API_METHODS["map()unmap()protect()map_region()unmap_region()protect_region()"]

GPTE_TRAIT --> GPTE_METHODS
GPTE_TRAIT --> PT64_STRUCT
PH_TRAIT --> PH_METHODS
PH_TRAIT --> PT64_STRUCT
PMD_TRAIT --> PMD_METHODS
PMD_TRAIT --> PT64_STRUCT
PT64_STRUCT --> API_METHODS
```

The `PagingMetaData` trait defines architecture constants, `GenericPTE` provides page table entry manipulation methods, and `PagingHandler` abstracts memory allocation and address translation for the operating system interface.

Sources: [README.md(L3)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/README.md#L3-L3) [CHANGELOG.md(L7)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/CHANGELOG.md#L7-L7) [CHANGELOG.md(L61 - L63)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/CHANGELOG.md#L61-L63)