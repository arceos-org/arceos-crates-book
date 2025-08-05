# Overview

> **Relevant source files**
> * [CHANGELOG.md](https://github.com/arceos-org/percpu/blob/89c8a54c/CHANGELOG.md)
> * [Cargo.toml](https://github.com/arceos-org/percpu/blob/89c8a54c/Cargo.toml)
> * [README.md](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md)

This document provides an overview of the **percpu** crate ecosystem, a Rust library system that provides efficient per-CPU data management for bare-metal and kernel development. The system consists of two main crates that work together to enable safe, efficient access to CPU-local data structures across multiple processor architectures.

The percpu system solves the fundamental problem of managing data that needs to be accessible per-CPU without synchronization overhead, critical for kernel and hypervisor development. For detailed API usage patterns, see [Basic Usage Examples](/arceos-org/percpu/2.2-basic-usage-examples). For implementation details of the code generation pipeline, see [Code Generation Pipeline](/arceos-org/percpu/3.3-code-generation-pipeline).

## System Architecture

The percpu ecosystem is built around two cooperating crates that provide compile-time code generation and runtime per-CPU data management:

```mermaid
flowchart TD
subgraph subGraph4["Architecture Registers"]
    X86_GS["x86_64: GS_BASE"]
    ARM_TPIDR["AArch64: TPIDR_EL1/EL2"]
    RISCV_GP["RISC-V: gp"]
    LOONG_R21["LoongArch: $r21"]
end
subgraph subGraph3["Memory Layout"]
    PERCPU_SECTION[".percpu section"]
    CPU_AREAS["Per-CPU data areasCPU 0, CPU 1, ..., CPU N"]
    LINKER_SCRIPT["_percpu_start_percpu_end_percpu_load_start"]
end
subgraph subGraph2["percpu Runtime Crate"]
    INIT_FUNC["init() -> usize"]
    INIT_REG["init_percpu_reg(cpu_id)"]
    AREA_ALLOC["percpu_area_base()"]
    REG_ACCESS["read_percpu_reg()write_percpu_reg()"]
end
subgraph subGraph1["percpu_macros Crate"]
    DEF_PERCPU["def_percpu proc macro"]
    CODE_GEN["Architecture-specificcode generation"]
    GENERATED["__PERCPU_CPU_IDCPU_ID_WRAPPERassembly blocks"]
end
subgraph subGraph0["Application Code"]
    USER_VAR["#[percpu::def_percpu]static CPU_ID: usize = 0"]
    USER_INIT["percpu::init()"]
    USER_ACCESS["CPU_ID.read_current()"]
end

AREA_ALLOC --> CPU_AREAS
ARM_TPIDR --> CPU_AREAS
CODE_GEN --> GENERATED
CPU_AREAS --> LINKER_SCRIPT
DEF_PERCPU --> CODE_GEN
GENERATED --> PERCPU_SECTION
GENERATED --> REG_ACCESS
INIT_FUNC --> AREA_ALLOC
INIT_REG --> ARM_TPIDR
INIT_REG --> LOONG_R21
INIT_REG --> RISCV_GP
INIT_REG --> X86_GS
LOONG_R21 --> CPU_AREAS
PERCPU_SECTION --> CPU_AREAS
REG_ACCESS --> ARM_TPIDR
REG_ACCESS --> LOONG_R21
REG_ACCESS --> RISCV_GP
REG_ACCESS --> X86_GS
RISCV_GP --> CPU_AREAS
USER_ACCESS --> GENERATED
USER_INIT --> INIT_FUNC
USER_INIT --> INIT_REG
USER_VAR --> DEF_PERCPU
X86_GS --> CPU_AREAS
```

**Sources: [README.md(L40 - L52)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L40-L52) [Cargo.toml(L4 - L7)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/Cargo.toml#L4-L7)**

## Code Generation and Runtime Flow

The system transforms user-defined per-CPU variables into architecture-specific implementations through a multi-stage process:

```mermaid
flowchart TD
subgraph subGraph3["Access Methods"]
    READ_CURRENT["read_current()"]
    WRITE_CURRENT["write_current()"]
    REMOTE_PTR["remote_ptr(cpu_id)"]
    REMOTE_REF["remote_ref(cpu_id)"]
end
subgraph subGraph2["Runtime Execution"]
    INIT_CALL["percpu::init()"]
    DETECT_CPUS["detect CPU count"]
    ALLOC_AREAS["allocate per-CPU areas"]
    COPY_TEMPLATE["copy .percpu templateto each CPU area"]
    SET_REGISTERS["init_percpu_reg(cpu_id)per CPU"]
end
subgraph subGraph1["Generated Artifacts"]
    STORAGE_VAR["__PERCPU_VARNAMEin .percpu section"]
    WRAPPER_STRUCT["VARNAME_WRAPPER"]
    PUBLIC_STATIC["VARNAME: VARNAME_WRAPPER"]
    ASM_BLOCKS["Architecture-specificinline assembly"]
end
subgraph subGraph0["Compile Time"]
    ATTR["#[percpu::def_percpu]"]
    PARSE["syn::parse_macro_input"]
    ARCH_DETECT["cfg(target_arch)"]
    QUOTE_GEN["quote! macro expansion"]
end

ALLOC_AREAS --> COPY_TEMPLATE
ARCH_DETECT --> QUOTE_GEN
ASM_BLOCKS --> READ_CURRENT
ASM_BLOCKS --> WRITE_CURRENT
ATTR --> PARSE
COPY_TEMPLATE --> SET_REGISTERS
DETECT_CPUS --> ALLOC_AREAS
INIT_CALL --> DETECT_CPUS
PARSE --> ARCH_DETECT
PUBLIC_STATIC --> READ_CURRENT
PUBLIC_STATIC --> REMOTE_PTR
PUBLIC_STATIC --> REMOTE_REF
PUBLIC_STATIC --> WRITE_CURRENT
QUOTE_GEN --> ASM_BLOCKS
QUOTE_GEN --> PUBLIC_STATIC
QUOTE_GEN --> STORAGE_VAR
QUOTE_GEN --> WRAPPER_STRUCT
SET_REGISTERS --> READ_CURRENT
SET_REGISTERS --> WRITE_CURRENT
```

**Sources: [README.md(L39 - L52)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L39-L52) [CHANGELOG.md(L8 - L12)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/CHANGELOG.md#L8-L12)**

## Platform Support Matrix

The percpu system provides cross-platform support through architecture-specific register handling:

|Architecture|Register Used|Assembly Instructions|Feature Support|
| --- | --- | --- | --- |
|x86_64|GS_BASE|mov gs:[offset], reg|Full support|
|AArch64|TPIDR_EL1/EL2|mrs/msrinstructions|EL1/EL2 modes|
|RISC-V|gp|mv gp, reg|32-bit and 64-bit|
|LoongArch|$r21|lu12i.w/ori|64-bit support|

The system adapts its code generation based on `cfg(target_arch)` directives and provides feature flags for specialized environments:

* **`sp-naive`**: Single-processor fallback using global variables
* **`preempt`**: Preemption-safe access with `NoPreemptGuard` integration
* **`arm-el2`**: Hypervisor mode support using `TPIDR_EL2`

**Sources: [README.md(L19 - L35)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L19-L35) [README.md(L69 - L79)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L69-L79)**

## Integration Requirements

### Linker Script Configuration

The system requires manual linker script modification to define the `.percpu` section layout:

```css
_percpu_start = .;
_percpu_end = _percpu_start + SIZEOF(.percpu);
.percpu 0x0 (NOLOAD) : AT(_percpu_start) {
    _percpu_load_start = .;
    *(.percpu .percpu.*)
    _percpu_load_end = .;
    . = _percpu_load_start + ALIGN(64) * CPU_NUM;
}
```

### Workspace Structure

The system is organized as a Cargo workspace containing:

* **`percpu`**: Runtime implementation and API
* **`percpu_macros`**: Procedural macro implementation

Both crates share common metadata and versioning through `workspace.package` configuration.

**Sources: [README.md(L54 - L67)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L54-L67) [Cargo.toml(L1 - L25)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/Cargo.toml#L1-L25)**

## Key API Entry Points

The primary user interface consists of:

* **`percpu::def_percpu`**: Procedural macro for defining per-CPU variables
* **`percpu::init()`**: Initialize per-CPU memory areas, returns CPU count
* **`percpu::init_percpu_reg(cpu_id)`**: Configure per-CPU register for specific CPU
* **`.read_current()`** / **`.write_current()`**: Access current CPU's data
* **`.remote_ptr(cpu_id)`** / **`.remote_ref(cpu_id)`**: Access remote CPU data

The system generates wrapper types that provide these methods while maintaining type safety and architecture-specific optimization.

**Sources: [README.md(L40 - L52)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L40-L52) [CHANGELOG.md(L8 - L12)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/CHANGELOG.md#L8-L12)**