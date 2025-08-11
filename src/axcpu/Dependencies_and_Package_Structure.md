# Dependencies and Package Structure

> **Relevant source files**
> * [Cargo.lock](https://github.com/arceos-org/axcpu/blob/b93d8fa3/Cargo.lock)

This document explains the dependency structure and package organization of the axcpu library. It covers the external crates that provide architecture-specific functionality, common utilities, and abstractions used across all supported architectures. For detailed information about specific architecture implementations, see the individual architecture sections ([x86_64](/arceos-org/axcpu/2-x86_64-architecture), [AArch64](/arceos-org/axcpu/3-aarch64-architecture), [RISC-V](/arceos-org/axcpu/4-risc-v-architecture), [LoongArch64](/arceos-org/axcpu/5-loongarch64-architecture)).

## Core Package Structure

The axcpu crate is organized as a single package that provides multi-architecture CPU abstraction through conditional compilation and architecture-specific dependencies. The library selectively includes dependencies based on the target architecture being compiled for.

### Main Dependency Categories

```mermaid
flowchart TD
subgraph subGraph4["axcpu Package Structure"]
    AXCPU["axcpu v0.1.1Main Package"]
    subgraph subGraph3["Build Support"]
        CFG_IF["cfg-if v1.0.1Conditional compilation"]
        STATIC_ASSERTIONS["static_assertions v1.1.0Compile-time checks"]
        LOG["log v0.4.27Logging facade"]
    end
    subgraph subGraph2["System Utilities"]
        PERCPU["percpu v0.2.0Per-CPU data structures"]
        LAZYINIT["lazyinit v0.2.2Lazy initialization"]
        LINKME["linkme v0.3.33Linker-based collections"]
        TOCK_REGISTERS["tock-registers v0.9.0Register field access"]
    end
    subgraph subGraph1["Memory Management"]
        MEMORY_ADDR["memory_addr v0.3.2Address type abstractions"]
        PAGE_TABLE_ENTRY["page_table_entry v0.5.3Page table entry types"]
        PAGE_TABLE_MULTI["page_table_multiarch v0.5.3Multi-arch page tables"]
    end
    subgraph subGraph0["Architecture Support Crates"]
        X86["x86 v0.52.0x86 CPU abstractions"]
        X86_64["x86_64 v0.15.2x86_64 specific features"]
        AARCH64["aarch64-cpu v10.0.0ARM64 register access"]
        RISCV["riscv v0.14.0RISC-V CSR and instructions"]
        LOONGARCH["loongArch64 v0.2.5LoongArch64 support"]
    end
end

AXCPU --> AARCH64
AXCPU --> CFG_IF
AXCPU --> LAZYINIT
AXCPU --> LINKME
AXCPU --> LOG
AXCPU --> LOONGARCH
AXCPU --> MEMORY_ADDR
AXCPU --> PAGE_TABLE_ENTRY
AXCPU --> PAGE_TABLE_MULTI
AXCPU --> PERCPU
AXCPU --> RISCV
AXCPU --> STATIC_ASSERTIONS
AXCPU --> TOCK_REGISTERS
AXCPU --> X86
AXCPU --> X86_64
```

Sources: [Cargo.lock(L21 - L39)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/Cargo.lock#L21-L39)

## Architecture-Specific Dependencies

Each supported architecture relies on specialized crates that provide low-level access to CPU features, registers, and instructions.

### x86/x86_64 Dependencies

```mermaid
flowchart TD
subgraph subGraph2["x86_64 Architecture Support"]
    X86_CRATE["x86 v0.52.0"]
    X86_64_CRATE["x86_64 v0.15.2"]
    subgraph subGraph1["x86_64 Dependencies"]
        BITFLAGS_2["bitflags v2.9.1"]
        VOLATILE["volatile v0.4.6"]
        RUSTVERSION["rustversion v1.0.21"]
    end
    subgraph subGraph0["x86 Dependencies"]
        BIT_FIELD["bit_field v0.10.2"]
        BITFLAGS_1["bitflags v1.3.2"]
        RAW_CPUID["raw-cpuid v10.7.0"]
    end
end

X86_64_CRATE --> BITFLAGS_2
X86_64_CRATE --> BIT_FIELD
X86_64_CRATE --> RUSTVERSION
X86_64_CRATE --> VOLATILE
X86_CRATE --> BITFLAGS_1
X86_CRATE --> BIT_FIELD
X86_CRATE --> RAW_CPUID
```

|Crate|Purpose|Key Features|
| --- | --- | --- |
|x86|32-bit x86 support|CPU ID, MSR access, I/O port operations|
|x86_64|64-bit extensions|Page table management, 64-bit registers, VirtAddr/PhysAddr|
|raw-cpuid|CPU feature detection|CPUID instruction wrapper|
|volatile|Memory access control|Prevents compiler optimizations on memory operations|

Sources: [Cargo.lock(L315 - L335)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/Cargo.lock#L315-L335)

### AArch64 Dependencies

```mermaid
flowchart TD
subgraph subGraph0["AArch64 Architecture Support"]
    AARCH64_CRATE["aarch64-cpu v10.0.0"]
    TOCK_REG["tock-registers v0.9.0"]
end

AARCH64_CRATE --> TOCK_REG
```

The `aarch64-cpu` crate provides register access patterns and system control for ARM64 processors, built on top of the `tock-registers` framework for type-safe register field manipulation.

Sources: [Cargo.lock(L6 - L12)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/Cargo.lock#L6-L12)

### RISC-V Dependencies

```mermaid
flowchart TD
subgraph subGraph1["RISC-V Architecture Support"]
    RISCV_CRATE["riscv v0.14.0"]
    subgraph subGraph0["RISC-V Dependencies"]
        CRITICAL_SECTION["critical-section v1.2.0"]
        EMBEDDED_HAL["embedded-hal v1.0.0"]
        PASTE["paste v1.0.15"]
        RISCV_MACROS["riscv-macros v0.2.0"]
        RISCV_PAC["riscv-pac v0.2.0"]
    end
end

RISCV_CRATE --> CRITICAL_SECTION
RISCV_CRATE --> EMBEDDED_HAL
RISCV_CRATE --> PASTE
RISCV_CRATE --> RISCV_MACROS
RISCV_CRATE --> RISCV_PAC
```

|Component|Purpose|
| --- | --- |
|riscv-macros|Procedural macros for CSR access|
|riscv-pac|Peripheral Access Crate for RISC-V|
|critical-section|Interrupt-safe critical sections|
|paste|Token pasting for macro generation|

Sources: [Cargo.lock(L229 - L239)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/Cargo.lock#L229-L239)

### LoongArch64 Dependencies

```mermaid
flowchart TD
subgraph subGraph1["LoongArch64 Architecture Support"]
    LOONGARCH_CRATE["loongArch64 v0.2.5"]
    subgraph subGraph0["LoongArch Dependencies"]
        BIT_FIELD_LA["bit_field v0.10.2"]
        BITFLAGS_LA["bitflags v2.9.1"]
    end
end

LOONGARCH_CRATE --> BITFLAGS_LA
LOONGARCH_CRATE --> BIT_FIELD_LA
```

The LoongArch64 support uses bit manipulation utilities for register and instruction field access.

Sources: [Cargo.lock(L120 - L127)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/Cargo.lock#L120-L127)

## Memory Management Dependencies

The memory management subsystem relies on a set of interconnected crates that provide address abstractions and page table management across architectures.

```mermaid
flowchart TD
subgraph subGraph2["Memory Management Dependency Chain"]
    PAGE_TABLE_ENTRY_CRATE["page_table_entry v0.5.3PTE abstractions"]
    PAGE_TABLE_MULTI_CRATE["page_table_multiarch v0.5.3Multi-arch page tables"]
    MEMORY_ADDR_CRATE["memory_addr v0.3.2VirtAddr, PhysAddr types"]
    subgraph subGraph1["Multi-arch Page Table Dependencies"]
        PTM_LOG["log"]
        PTM_MEMORY_ADDR["memory_addr"]
        PTM_PAGE_TABLE_ENTRY["page_table_entry"]
        PTM_RISCV["riscv v0.12.1"]
        PTM_X86["x86"]
    end
    subgraph subGraph0["Page Table Entry Dependencies"]
        PTE_AARCH64["aarch64-cpu"]
        PTE_X86_64["x86_64"]
        PTE_BITFLAGS["bitflags v2.9.1"]
        PTE_MEMORY_ADDR["memory_addr"]
    end
end

MEMORY_ADDR_CRATE --> PAGE_TABLE_ENTRY_CRATE
PAGE_TABLE_ENTRY_CRATE --> PAGE_TABLE_MULTI_CRATE
PAGE_TABLE_ENTRY_CRATE --> PTE_AARCH64
PAGE_TABLE_ENTRY_CRATE --> PTE_BITFLAGS
PAGE_TABLE_ENTRY_CRATE --> PTE_MEMORY_ADDR
PAGE_TABLE_ENTRY_CRATE --> PTE_X86_64
PAGE_TABLE_MULTI_CRATE --> PTM_LOG
PAGE_TABLE_MULTI_CRATE --> PTM_MEMORY_ADDR
PAGE_TABLE_MULTI_CRATE --> PTM_PAGE_TABLE_ENTRY
PAGE_TABLE_MULTI_CRATE --> PTM_RISCV
PAGE_TABLE_MULTI_CRATE --> PTM_X86
```

Sources: [Cargo.lock(L130 - L158)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/Cargo.lock#L130-L158)

## System Utility Dependencies

### Per-CPU Data Management

```mermaid
flowchart TD
subgraph subGraph2["percpu Crate Structure"]
    PERCPU_CRATE["percpu v0.2.0Per-CPU data structures"]
    subgraph subGraph1["Macro Dependencies"]
        PROC_MACRO2["proc-macro2 v1.0.95"]
        QUOTE["quote v1.0.40"]
        SYN["syn v2.0.104"]
    end
    subgraph subGraph0["percpu Dependencies"]
        PERCPU_CFG_IF["cfg-if v1.0.1"]
        PERCPU_MACROS["percpu_macros v0.2.0"]
        PERCPU_SPIN["spin v0.9.8"]
        PERCPU_X86["x86 v0.52.0"]
    end
end

PERCPU_CRATE --> PERCPU_CFG_IF
PERCPU_CRATE --> PERCPU_MACROS
PERCPU_CRATE --> PERCPU_SPIN
PERCPU_CRATE --> PERCPU_X86
PERCPU_MACROS --> PROC_MACRO2
PERCPU_MACROS --> QUOTE
PERCPU_MACROS --> SYN
```

The `percpu` crate provides architecture-aware per-CPU data structures with x86-specific optimizations for CPU-local storage access.

Sources: [Cargo.lock(L167 - L187)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/Cargo.lock#L167-L187)

### Linker-Based Collections

The `linkme` crate enables compile-time collection of distributed static data, used for registering trap handlers and other system components across architecture modules.

|Crate|Purpose|Dependencies|
| --- | --- | --- |
|linkme|Distributed static collections|linkme-impl|
|linkme-impl|Implementation macros|proc-macro2,quote,syn|

Sources: [Cargo.lock(L84 - L101)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/Cargo.lock#L84-L101)

## Conditional Compilation Structure

The axcpu library uses feature-based compilation to include only the dependencies and code paths relevant to the target architecture.

```

```

Sources: [Cargo.lock(L60 - L63)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/Cargo.lock#L60-L63)

## Build-Time Dependencies

### Static Assertions and Verification

The `static_assertions` crate provides compile-time verification of type sizes, alignment requirements, and other invariants critical for low-level CPU context management.

### Procedural Macro Infrastructure

|Macro Crate|Purpose|Used By|
| --- | --- | --- |
|proc-macro2|Procedural macro toolkit|linkme-impl,percpu_macros,riscv-macros|
|quote|Code generation helper|All macro implementations|
|syn|Rust syntax parsing|All macro implementations|

Sources: [Cargo.lock(L190 - L294)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/Cargo.lock#L190-L294)

## Version Constraints and Compatibility

The dependency graph maintains compatibility across architecture-specific crates through careful version selection:

* **Bitflags**: Uses both v1.3.2 (for x86) and v2.9.1 (for newer crates)
* **RISC-V**: Uses v0.14.0 for axcpu, v0.12.1 for page table compatibility
* **Register Access**: Standardized on `tock-registers` v0.9.0 for AArch64

This structure ensures that each architecture module can use the most appropriate version of its dependencies while maintaining overall system compatibility.

Sources: [Cargo.lock(L1 - L336)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/Cargo.lock#L1-L336)