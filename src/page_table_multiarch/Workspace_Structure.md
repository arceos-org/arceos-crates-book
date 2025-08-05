# Workspace Structure

> **Relevant source files**
> * [Cargo.toml](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.toml)
> * [page_table_entry/Cargo.toml](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/Cargo.toml)
> * [page_table_multiarch/Cargo.toml](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/Cargo.toml)

This document explains the two-crate workspace structure of the `page_table_multiarch` repository and how the crates interact to provide multi-architecture page table functionality. It covers the workspace configuration, crate responsibilities, dependency management, and conditional compilation strategy.

For detailed information about the individual crate implementations, see [page_table_multiarch Crate](/arceos-org/page_table_multiarch/2.1-page_table_multiarch-crate) and [page_table_entry Crate](/arceos-org/page_table_multiarch/2.2-page_table_entry-crate).

## Workspace Overview

The `page_table_multiarch` repository is organized as a Cargo workspace containing two complementary crates that provide a layered approach to page table management across multiple hardware architectures.

### Workspace Definition

The workspace is defined in the root [Cargo.toml(L1 - L7)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.toml#L1-L7) with two member crates:

|Crate|Purpose|Role|
| --- | --- | --- |
|page_table_multiarch|High-level page table abstractions|Main API providingPageTable64and unified interface|
|page_table_entry|Low-level page table entries|Architecture-specific PTE implementations andGenericPTEtrait|

**Workspace Structure Diagram**

```mermaid
flowchart TD
subgraph subGraph2["page_table_multiarch Workspace"]
    ROOT["Cargo.toml(Workspace Root)"]
    subgraph subGraph1["page_table_entry Crate"]
        PTE["page_table_entryPage table entry definitions"]
        PTE_CARGO["page_table_entry/Cargo.toml"]
    end
    subgraph subGraph0["page_table_multiarch Crate"]
        PTM_CARGO["page_table_multiarch/Cargo.toml"]
        PTM["page_table_multiarchGeneric page table structures"]
    end
end

PTE_CARGO --> PTE
PTM --> PTE
PTM_CARGO --> PTM
ROOT --> PTE_CARGO
ROOT --> PTM_CARGO
```

*Sources: [Cargo.toml(L4 - L7)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.toml#L4-L7) [page_table_multiarch/Cargo.toml(L2 - L3)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/Cargo.toml#L2-L3) [page_table_entry/Cargo.toml(L2 - L3)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/Cargo.toml#L2-L3)*

### Shared Workspace Configuration

The workspace defines common package metadata in [Cargo.toml(L9 - L19)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.toml#L9-L19) that is inherited by both member crates:

* **Version**: `0.5.3` - synchronized across all crates
* **Edition**: `2024` - uses latest Rust edition
* **Categories**: `["os", "hardware-support", "memory-management", "no-std"]`
* **Keywords**: `["arceos", "paging", "page-table", "virtual-memory"]`
* **Rust Version**: `1.85` - minimum supported Rust version

*Sources: [Cargo.toml(L9 - L19)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.toml#L9-L19) [page_table_multiarch/Cargo.toml(L5 - L13)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/Cargo.toml#L5-L13) [page_table_entry/Cargo.toml(L5 - L13)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/Cargo.toml#L5-L13)*

## Crate Interaction Architecture

The two crates form a layered architecture where `page_table_multiarch` provides high-level abstractions while depending on `page_table_entry` for low-level functionality.

**Crate Dependency and Interaction Diagram**

```mermaid
flowchart TD
subgraph subGraph3["External Dependencies"]
    MEMORY_ADDR["memory_addr crateAddress types"]
    LOG["log crateLogging"]
    BITFLAGS["bitflags crateFlag manipulation"]
    ARCH_SPECIFIC["x86, riscv, aarch64-cpux86_64 crates"]
end
subgraph subGraph2["page_table_entry Crate"]
    PTE_TRAIT["GenericPTE trait"]
    PTE_IMPL["X64PTE, A64PTERv64PTE, LA64PTE"]
    PTE_FLAGS["MappingFlagsArchitecture conversion"]
end
subgraph subGraph1["page_table_multiarch Crate"]
    PTM_API["PageTable64"]
    PTM_TRAITS["PagingMetaDataPagingHandler traits"]
    PTM_IMPL["Architecture-specificPageTable implementations"]
end
subgraph subGraph0["Application Layer"]
    APP["Operating SystemsHypervisorsKernel Code"]
end

APP --> PTM_API
PTE_IMPL --> ARCH_SPECIFIC
PTE_IMPL --> BITFLAGS
PTE_IMPL --> PTE_FLAGS
PTE_TRAIT --> MEMORY_ADDR
PTE_TRAIT --> PTE_FLAGS
PTM_API --> LOG
PTM_API --> MEMORY_ADDR
PTM_API --> PTM_IMPL
PTM_API --> PTM_TRAITS
PTM_IMPL --> ARCH_SPECIFIC
PTM_IMPL --> PTE_IMPL
PTM_TRAITS --> PTE_TRAIT
```

*Sources: [page_table_multiarch/Cargo.toml(L15 - L18)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/Cargo.toml#L15-L18) [page_table_entry/Cargo.toml(L18 - L20)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/Cargo.toml#L18-L20)*

### Dependency Relationship

The `page_table_multiarch` crate explicitly depends on `page_table_entry` as defined in [page_table_multiarch/Cargo.toml(L18)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/Cargo.toml#L18-L18):

```
page_table_entry = { path = "../page_table_entry", version = "0.5.2" }
```

This creates a clear separation of concerns:

* **High-level operations** (page table walking, mapping, unmapping) in `page_table_multiarch`
* **Low-level PTE manipulation** (bit operations, flag conversions) in `page_table_entry`

*Sources: [page_table_multiarch/Cargo.toml(L18)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/Cargo.toml#L18-L18)*

## Architecture-Specific Dependencies

Both crates use conditional compilation to include only the necessary architecture-specific dependencies based on the target platform.

### page_table_multiarch Dependencies

**Conditional Dependencies Diagram**

```mermaid
flowchart TD
subgraph subGraph0["page_table_multiarch Dependencies"]
    CORE["Core Dependencieslog, memory_addrpage_table_entry"]
    X86_TARGET["cfg(target_arch = x86_64)"]
    X86_DEP["x86 = 0.52"]
    RISCV_TARGET["cfg(target_arch = riscv32/64)"]
    RISCV_DEP["riscv = 0.12"]
    DOC_TARGET["cfg(doc)"]
    ALL_DEP["All arch dependenciesfor documentation"]
end

CORE --> DOC_TARGET
CORE --> RISCV_TARGET
CORE --> X86_TARGET
DOC_TARGET --> ALL_DEP
RISCV_TARGET --> RISCV_DEP
X86_TARGET --> X86_DEP
```

*Sources: [page_table_multiarch/Cargo.toml(L20 - L24)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/Cargo.toml#L20-L24) [page_table_multiarch/Cargo.toml(L26 - L27)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/Cargo.toml#L26-L27)*

### page_table_entry Dependencies

The `page_table_entry` crate includes more architecture-specific dependencies:

|Target Architecture|Dependencies|Purpose|
| --- | --- | --- |
|x86_64|x86_64 = "0.15.2"|x86-64 specific register and instruction access|
|aarch64|aarch64-cpu = "10.0"|ARM64 system register and instruction access|
|riscv32/riscv64|None (built-in)|RISC-V support uses standard library features|
|loongarch64|None (built-in)|LoongArch support uses standard library features|

**page_table_entry Architecture Dependencies**

```mermaid
flowchart TD
subgraph subGraph2["page_table_entry Conditional Dependencies"]
    PTE_CORE["Core Dependenciesbitflags, memory_addr"]
    subgraph subGraph1["Feature Flags"]
        ARM_EL2["arm-el2 featureARM Exception Level 2"]
    end
    subgraph subGraph0["Architecture Dependencies"]
        AARCH64_CFG["cfg(target_arch = aarch64)"]
        X86_64_CFG["cfg(target_arch = x86_64)"]
        DOC_CFG["cfg(doc)"]
        AARCH64_DEP["aarch64-cpu = 10.0"]
        X86_64_DEP["x86_64 = 0.15.2"]
        DOC_DEP["All dependencies"]
    end
end

AARCH64_CFG --> AARCH64_DEP
AARCH64_DEP --> ARM_EL2
DOC_CFG --> DOC_DEP
PTE_CORE --> AARCH64_CFG
PTE_CORE --> DOC_CFG
PTE_CORE --> X86_64_CFG
X86_64_CFG --> X86_64_DEP
```

*Sources: [page_table_entry/Cargo.toml(L22 - L26)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/Cargo.toml#L22-L26) [page_table_entry/Cargo.toml(L15 - L16)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/Cargo.toml#L15-L16)*

## Documentation Configuration

Both crates include special configuration for documentation generation using `docs.rs`:

* **Documentation builds** include all architecture dependencies via `cfg(doc)`
* **Rustc arguments** add `--cfg doc` for conditional compilation during docs generation
* This ensures complete documentation coverage across all supported architectures

The configuration in [page_table_multiarch/Cargo.toml(L26 - L27)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/Cargo.toml#L26-L27) and [page_table_entry/Cargo.toml(L28 - L29)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/Cargo.toml#L28-L29) enables this behavior.

*Sources: [page_table_multiarch/Cargo.toml(L26 - L27)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/Cargo.toml#L26-L27) [page_table_entry/Cargo.toml(L28 - L29)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/Cargo.toml#L28-L29)*

## Workspace Benefits

The two-crate structure provides several architectural advantages:

1. **Separation of Concerns**: High-level page table operations vs. low-level PTE manipulation
2. **Conditional Compilation**: Architecture-specific dependencies are isolated and only included when needed
3. **Reusability**: The `page_table_entry` crate can be used independently for PTE operations
4. **Testing**: Each layer can be tested independently with appropriate mocking
5. **Documentation**: Clear API boundaries make the system easier to understand and document

*Sources: [Cargo.toml(L1 - L19)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.toml#L1-L19) [page_table_multiarch/Cargo.toml(L1 - L28)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/Cargo.toml#L1-L28) [page_table_entry/Cargo.toml(L1 - L29)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/Cargo.toml#L1-L29)*