# Architecture Support

> **Relevant source files**
> * [page_table_entry/src/arch/mod.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/mod.rs)
> * [page_table_multiarch/src/arch/mod.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/mod.rs)

This document provides an overview of how the `page_table_multiarch` library supports multiple processor architectures through a unified abstraction layer. It covers the conditional compilation strategy, supported architectures, and the trait-based abstraction mechanism that enables architecture-independent page table management.

For detailed implementation specifics of individual architectures, see [x86_64 Support](/arceos-org/page_table_multiarch/4.1-x86_64-support), [AArch64 Support](/arceos-org/page_table_multiarch/4.2-aarch64-support), [RISC-V Support](/arceos-org/page_table_multiarch/4.3-risc-v-support), and [LoongArch64 Support](/arceos-org/page_table_multiarch/4.4-loongarch64-support).

## Conditional Compilation Strategy

The library uses Rust's conditional compilation features to include only the relevant architecture-specific code for the target platform. This approach minimizes binary size and compile time while maintaining support for multiple architectures in a single codebase.

### Compilation Configuration

The architecture modules are conditionally compiled based on the target architecture:

```

```

Sources: [page_table_entry/src/arch/mod.rs(L1 - L12)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/mod.rs#L1-L12) [page_table_multiarch/src/arch/mod.rs(L1 - L12)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/mod.rs#L1-L12)

The `doc` condition ensures all architecture modules are available during documentation generation, allowing comprehensive API documentation regardless of the build target.

## Supported Architectures Overview

The library supports four major processor architectures, each with distinct paging characteristics and implementations:

|Architecture|Page Table Levels|Virtual Address Width|Physical Address Width|Key Features|
| --- | --- | --- | --- | --- |
|x86_64|4 levels (PML4)|48 bits|52 bits|Legacy support, complex permissions|
|AArch64|4 levels (VMSAv8-64)|48 bits|48 bits|Memory attributes, EL2 support|
|RISC-V|3 levels (Sv39) / 4 levels (Sv48)|39/48 bits|Variable|Multiple page table formats|
|LoongArch64|4 levels|Variable|Variable|Page Walk Controller (PWC)|

### Architecture Module Structure

```mermaid
flowchart TD
subgraph subGraph2["Concrete Types"]
    X64PT["X64PageTable"]
    A64PT["A64PageTable"]
    SV39PT["Sv39PageTable"]
    SV48PT["Sv48PageTable"]
    LA64PT["LA64PageTable"]
    X64PTE["X64PTE"]
    A64PTE["A64PTE"]
    RV64PTE["Rv64PTE"]
    LA64PTE["LA64PTE"]
end
subgraph subGraph1["page_table_entry Crate"]
    PTE_ARCH["src/arch/"]
    PTE_X86["src/arch/x86_64.rs"]
    PTE_ARM["src/arch/aarch64.rs"]
    PTE_RV["src/arch/riscv.rs"]
    PTE_LA["src/arch/loongarch64.rs"]
end
subgraph subGraph0["page_table_multiarch Crate"]
    PTA_ARCH["src/arch/"]
    PTA_X86["src/arch/x86_64.rs"]
    PTA_ARM["src/arch/aarch64.rs"]
    PTA_RV["src/arch/riscv.rs"]
    PTA_LA["src/arch/loongarch64.rs"]
end

A64PT --> A64PTE
LA64PT --> LA64PTE
PTA_ARCH --> PTA_ARM
PTA_ARCH --> PTA_LA
PTA_ARCH --> PTA_RV
PTA_ARCH --> PTA_X86
PTA_ARM --> A64PT
PTA_LA --> LA64PT
PTA_RV --> SV39PT
PTA_RV --> SV48PT
PTA_X86 --> X64PT
PTE_ARCH --> PTE_ARM
PTE_ARCH --> PTE_LA
PTE_ARCH --> PTE_RV
PTE_ARCH --> PTE_X86
PTE_ARM --> A64PTE
PTE_LA --> LA64PTE
PTE_RV --> RV64PTE
PTE_X86 --> X64PTE
SV39PT --> RV64PTE
SV48PT --> RV64PTE
X64PT --> X64PTE
```

Sources: [page_table_entry/src/arch/mod.rs(L1 - L12)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/mod.rs#L1-L12) [page_table_multiarch/src/arch/mod.rs(L1 - L12)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/mod.rs#L1-L12)

## Architecture Abstraction Mechanism

The multi-architecture support is achieved through a trait-based abstraction layer that defines common interfaces while allowing architecture-specific implementations.

### Core Abstraction Traits

```mermaid
flowchart TD
subgraph subGraph5["Architecture Implementations"]
    subgraph LoongArch64["LoongArch64"]
        LA64MD["LA64MetaData"]
        LA64PTE_IMPL["LA64PTE"]
    end
    subgraph RISC-V["RISC-V"]
        SV39MD["Sv39MetaData"]
        SV48MD["Sv48MetaData"]
        RV64PTE_IMPL["Rv64PTE"]
    end
    subgraph AArch64["AArch64"]
        A64MD["A64PagingMetaData"]
        A64PTE_IMPL["A64PTE"]
    end
    subgraph x86_64["x86_64"]
        X64MD["X64PagingMetaData"]
        X64PTE_IMPL["X64PTE"]
    end
end
subgraph subGraph0["Generic Abstractions"]
    PT64["PageTable64<M, PTE, H>"]
    PMD["PagingMetaData trait"]
    GPTE["GenericPTE trait"]
    PH["PagingHandler trait"]
    MF["MappingFlags"]
end

GPTE --> A64PTE_IMPL
GPTE --> LA64PTE_IMPL
GPTE --> MF
GPTE --> RV64PTE_IMPL
GPTE --> X64PTE_IMPL
PMD --> A64MD
PMD --> LA64MD
PMD --> SV39MD
PMD --> SV48MD
PMD --> X64MD
PT64 --> GPTE
PT64 --> PH
PT64 --> PMD
```

Sources: [page_table_entry/src/arch/mod.rs(L1 - L12)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/mod.rs#L1-L12) [page_table_multiarch/src/arch/mod.rs(L1 - L12)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/mod.rs#L1-L12)

### Generic Type Parameters

The `PageTable64<M, PTE, H>` type uses three generic parameters to achieve architecture independence:

* **M**: Implements `PagingMetaData` - provides architecture-specific constants like page sizes, address widths, and level counts
* **PTE**: Implements `GenericPTE` - handles page table entry creation, flag conversion, and address extraction
* **H**: Implements `PagingHandler` - manages frame allocation and deallocation through OS-specific interfaces

## Integration Between Crates

The workspace structure separates concerns between high-level page table management and low-level entry manipulation:

### Crate Dependency Flow

```mermaid
flowchart TD
subgraph subGraph2["External Dependencies"]
    MEMORY_ADDR["memory_addr"]
    HW_CRATES["Architecture-specifichardware crates"]
end
subgraph page_table_entry["page_table_entry"]
    LIB_PTE["lib.rs"]
    ARCH_PTE["arch/ modules"]
    PTE_TYPES["PTE Types:X64PTEA64PTERv64PTELA64PTE"]
    TRAITS["GenericPTE traitMappingFlags"]
end
subgraph page_table_multiarch["page_table_multiarch"]
    LIB_PTM["lib.rs"]
    BITS64["bits64.rs"]
    ARCH_PTM["arch/ modules"]
    PT_TYPES["Page Table Types:X64PageTableA64PageTableSv39PageTableSv48PageTableLA64PageTable"]
end

ARCH_PTE --> PTE_TYPES
ARCH_PTM --> PT_TYPES
LIB_PTE --> ARCH_PTE
LIB_PTE --> TRAITS
LIB_PTM --> ARCH_PTM
LIB_PTM --> BITS64
PTE_TYPES --> HW_CRATES
PTE_TYPES --> MEMORY_ADDR
PT_TYPES --> PTE_TYPES
PT_TYPES --> TRAITS
```

Sources: [page_table_entry/src/arch/mod.rs(L1 - L12)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/mod.rs#L1-L12) [page_table_multiarch/src/arch/mod.rs(L1 - L12)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/mod.rs#L1-L12)

This separation allows the `page_table_multiarch` crate to focus on high-level page table operations while `page_table_entry` handles the architecture-specific details of page table entry formats and flag mappings. The conditional compilation ensures that only the necessary code for the target architecture is included in the final binary.