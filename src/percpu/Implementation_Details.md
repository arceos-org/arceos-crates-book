# Implementation Details

> **Relevant source files**
> * [percpu/src/imp.rs](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs)
> * [percpu_macros/src/arch.rs](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs)
> * [percpu_macros/src/lib.rs](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs)

This page provides an overview of the internal implementation details of the percpu crate ecosystem for maintainers and advanced users. It covers the core implementation strategies, key code entities, and how the compile-time macro generation integrates with the runtime per-CPU data management system.

For detailed architecture-specific code generation, see [Architecture-Specific Code Generation](/arceos-org/percpu/5.1-architecture-specific-code-generation). For the simplified single-CPU implementation, see [Naive Implementation](/arceos-org/percpu/5.2-naive-implementation). For low-level memory management specifics, see [Memory Management Internals](/arceos-org/percpu/5.3-memory-management-internals).

## Core Implementation Strategy

The percpu system implements per-CPU data management through a two-phase approach: compile-time code generation via procedural macros and runtime memory area management. The system places all per-CPU variables in a special `.percpu` linker section, then creates per-CPU memory areas by copying this template section for each CPU.

```mermaid
flowchart TD
subgraph subGraph2["Linker Integration"]
    PERCPU_SEC[".percpu section"]
    PERCPU_START["_percpu_start"]
    PERCPU_END["_percpu_end"]
    LOAD_START["_percpu_load_start"]
    LOAD_END["_percpu_load_end"]
end
subgraph subGraph1["Runtime Phase"]
    INIT_FUNC["init()"]
    AREA_ALLOC["percpu_area_base()"]
    REG_INIT["init_percpu_reg()"]
    READ_REG["read_percpu_reg()"]
end
subgraph subGraph0["Compile-Time Phase"]
    DEF_PERCPU["def_percpu macro"]
    GEN_CODE["Code Generation Pipeline"]
    INNER_SYM["_PERCPU* symbols"]
    WRAPPER["*_WRAPPER structs"]
end

AREA_ALLOC --> REG_INIT
DEF_PERCPU --> GEN_CODE
GEN_CODE --> INNER_SYM
GEN_CODE --> WRAPPER
INIT_FUNC --> AREA_ALLOC
INNER_SYM --> PERCPU_SEC
LOAD_END --> AREA_ALLOC
LOAD_START --> AREA_ALLOC
PERCPU_END --> AREA_ALLOC
PERCPU_SEC --> INIT_FUNC
PERCPU_START --> AREA_ALLOC
REG_INIT --> READ_REG
WRAPPER --> AREA_ALLOC
WRAPPER --> READ_REG
```

Sources: [percpu_macros/src/lib.rs(L54 - L262)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L54-L262) [percpu/src/imp.rs(L1 - L179)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L1-L179)

## Runtime Implementation Architecture

The runtime implementation in `imp.rs` manages per-CPU memory areas and provides architecture-specific register access. The core functions handle initialization, memory layout calculation, and register manipulation across different CPU architectures.

```mermaid
flowchart TD
subgraph subGraph4["x86 Self-Pointer"]
    SELF_PTR["SELF_PTR: def_percpu static"]
end
subgraph subGraph3["Architecture-Specific Registers"]
    X86_GS["x86_64: IA32_GS_BASE"]
    ARM_TPIDR["aarch64: TPIDR_EL1/EL2"]
    RISCV_GP["riscv: gp register"]
    LOONG_R21["loongarch: $r21 register"]
end
subgraph subGraph2["Register Management"]
    READ_REG["read_percpu_reg()"]
    WRITE_REG["write_percpu_reg()"]
    INIT_REG["init_percpu_reg()"]
end
subgraph subGraph1["Memory Layout Functions"]
    AREA_SIZE["percpu_area_size()"]
    AREA_NUM["percpu_area_num()"]
    AREA_BASE["percpu_area_base(cpu_id)"]
    ALIGN_UP["align_up_64()"]
end
subgraph subGraph0["Initialization Functions"]
    INIT["init()"]
    IS_INIT["IS_INIT: AtomicBool"]
    PERCPU_AREA_BASE_STATIC["PERCPU_AREA_BASE: Once"]
end

AREA_BASE --> ALIGN_UP
AREA_NUM --> ALIGN_UP
AREA_SIZE --> ALIGN_UP
INIT --> AREA_BASE
INIT --> AREA_SIZE
INIT --> IS_INIT
INIT --> PERCPU_AREA_BASE_STATIC
INIT_REG --> AREA_BASE
INIT_REG --> WRITE_REG
READ_REG --> ARM_TPIDR
READ_REG --> LOONG_R21
READ_REG --> RISCV_GP
READ_REG --> X86_GS
WRITE_REG --> ARM_TPIDR
WRITE_REG --> LOONG_R21
WRITE_REG --> RISCV_GP
WRITE_REG --> X86_GS
X86_GS --> SELF_PTR
```

Sources: [percpu/src/imp.rs(L3 - L179)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L3-L179)

## Compile-Time Code Generation Architecture

The macro system in `percpu_macros` transforms user-defined per-CPU variables into architecture-optimized access code. The `def_percpu` macro generates wrapper structs with methods for safe and unsafe access patterns.

```mermaid
flowchart TD
subgraph subGraph3["Feature Configuration"]
    SP_NAIVE_FEATURE["sp-naive feature"]
    PREEMPT_FEATURE["preempt feature"]
    ARM_EL2_FEATURE["arm-el2 feature"]
    NO_PREEMPT_GUARD["NoPreemptGuard"]
end
subgraph subGraph2["Generated Code Entities"]
    INNER_SYMBOL["_PERCPU{name}"]
    WRAPPER_STRUCT["{name}_WRAPPER"]
    WRAPPER_STATIC["{name}: {name}_WRAPPER"]
    OFFSET_METHOD["offset()"]
    CURRENT_PTR_METHOD["current_ptr()"]
    WITH_CURRENT_METHOD["with_current()"]
    READ_WRITE_METHODS["read/write_current*()"]
    REMOTE_METHODS["remote_*_raw()"]
end
subgraph subGraph1["Code Generation Functions"]
    GEN_OFFSET["arch::gen_offset()"]
    GEN_CURRENT_PTR["arch::gen_current_ptr()"]
    GEN_READ_RAW["arch::gen_read_current_raw()"]
    GEN_WRITE_RAW["arch::gen_write_current_raw()"]
end
subgraph subGraph0["Input Processing"]
    DEF_PERCPU_ATTR["def_percpu attribute"]
    ITEM_STATIC["ItemStatic AST"]
    PARSE_INPUT["syn::parse_macro_input"]
end

ARM_EL2_FEATURE --> GEN_CURRENT_PTR
DEF_PERCPU_ATTR --> PARSE_INPUT
GEN_CURRENT_PTR --> WRAPPER_STRUCT
GEN_OFFSET --> INNER_SYMBOL
GEN_READ_RAW --> READ_WRITE_METHODS
GEN_WRITE_RAW --> READ_WRITE_METHODS
ITEM_STATIC --> PARSE_INPUT
NO_PREEMPT_GUARD --> READ_WRITE_METHODS
NO_PREEMPT_GUARD --> WITH_CURRENT_METHOD
PARSE_INPUT --> GEN_CURRENT_PTR
PARSE_INPUT --> GEN_OFFSET
PARSE_INPUT --> GEN_READ_RAW
PARSE_INPUT --> GEN_WRITE_RAW
PREEMPT_FEATURE --> NO_PREEMPT_GUARD
SP_NAIVE_FEATURE --> GEN_OFFSET
WRAPPER_STATIC --> WRAPPER_STRUCT
WRAPPER_STRUCT --> CURRENT_PTR_METHOD
WRAPPER_STRUCT --> OFFSET_METHOD
WRAPPER_STRUCT --> READ_WRITE_METHODS
WRAPPER_STRUCT --> REMOTE_METHODS
WRAPPER_STRUCT --> WITH_CURRENT_METHOD
```

Sources: [percpu_macros/src/lib.rs(L66 - L252)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L66-L252) [percpu_macros/src/arch.rs(L1 - L264)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L1-L264)

## Architecture-Specific Code Generation

The system generates different assembly code for each supported architecture to access per-CPU data efficiently. Each architecture uses different registers and instruction sequences for optimal performance.

|Architecture|Register|Offset Calculation|Access Pattern|
| --- | --- | --- | --- |
|x86_64|GS_BASE(IA32_GS_BASE)|offset symbol|mov gs:[offset VAR]|
|AArch64|TPIDR_EL1/TPIDR_EL2|#:abs_g0_nc:symbol|mrs TPIDR_ELx+ offset|
|RISC-V|gpregister|%hi(symbol)+%lo(symbol)|lui+addi+gp|
|LoongArch|$r21register|%abs_hi20+%abs_lo12|lu12i.w+ori+$r21|

```mermaid
flowchart TD
subgraph subGraph5["LoongArch Assembly"]
    LOONG_OFFSET["lu12i.w %abs_hi20 + ori %abs_lo12"]
    LOONG_R21_READ["move {}, $r21"]
    LOONG_R21_WRITE["move $r21, {}"]
    LOONG_LOAD_STORE["ldx./stx.with $r21"]
end
subgraph subGraph4["RISC-V Assembly"]
    RISCV_OFFSET["lui %hi + addi %lo"]
    RISCV_GP_READ["mv {}, gp"]
    RISCV_GP_WRITE["mv gp, {}"]
    RISCV_LOAD_STORE["ld/sd with gp offset"]
end
subgraph subGraph3["AArch64 Assembly"]
    ARM_OFFSET["movz #:abs_g0_nc:{VAR}"]
    ARM_TPIDR_READ["mrs TPIDR_EL1/EL2"]
    ARM_TPIDR_WRITE["msr TPIDR_EL1/EL2"]
end
subgraph subGraph2["x86_64 Assembly"]
    X86_OFFSET["mov {0:e}, offset {VAR}"]
    X86_READ["mov gs:[offset {VAR}]"]
    X86_WRITE["mov gs:[offset {VAR}], value"]
    X86_GS_BASE["IA32_GS_BASE MSR"]
end
subgraph subGraph1["Code Generation Functions"]
    GEN_OFFSET_IMPL["gen_offset()"]
    GEN_CURRENT_PTR_IMPL["gen_current_ptr()"]
    GEN_READ_IMPL["gen_read_current_raw()"]
    GEN_WRITE_IMPL["gen_write_current_raw()"]
end
subgraph subGraph0["Architecture Detection"]
    TARGET_ARCH["cfg!(target_arch)"]
    X86_64["x86_64"]
    AARCH64["aarch64"]
    RISCV["riscv32/riscv64"]
    LOONGARCH["loongarch64"]
end

AARCH64 --> GEN_OFFSET_IMPL
GEN_CURRENT_PTR_IMPL --> ARM_TPIDR_READ
GEN_CURRENT_PTR_IMPL --> LOONG_R21_READ
GEN_CURRENT_PTR_IMPL --> RISCV_GP_READ
GEN_OFFSET_IMPL --> ARM_OFFSET
GEN_OFFSET_IMPL --> LOONG_OFFSET
GEN_OFFSET_IMPL --> RISCV_OFFSET
GEN_OFFSET_IMPL --> X86_OFFSET
GEN_READ_IMPL --> LOONG_LOAD_STORE
GEN_READ_IMPL --> RISCV_LOAD_STORE
GEN_READ_IMPL --> X86_READ
GEN_WRITE_IMPL --> LOONG_LOAD_STORE
GEN_WRITE_IMPL --> RISCV_LOAD_STORE
GEN_WRITE_IMPL --> X86_WRITE
LOONGARCH --> GEN_OFFSET_IMPL
RISCV --> GEN_OFFSET_IMPL
TARGET_ARCH --> AARCH64
TARGET_ARCH --> LOONGARCH
TARGET_ARCH --> RISCV
TARGET_ARCH --> X86_64
```

Sources: [percpu_macros/src/arch.rs(L15 - L264)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L15-L264)

## Integration Between Runtime and Macros

The compile-time macros and runtime functions work together through shared conventions and generated code that calls runtime functions. The macros generate code that uses runtime functions for memory calculations and remote access.

```mermaid
flowchart TD
subgraph subGraph3["Memory Layout"]
    TEMPLATE_AREA["Template in .percpu"]
    CPU0_AREA["CPU 0 Data Area"]
    CPU1_AREA["CPU 1 Data Area"]
    CPUN_AREA["CPU N Data Area"]
end
subgraph subGraph2["Shared Data Structures"]
    PERCPU_SECTION[".percpu section"]
    LINKER_SYMBOLS["_percpu_start/_percpu_end"]
    INNER_SYMBOLS["_PERCPU* symbols"]
    CPU_REGISTERS["Architecture registers"]
end
subgraph subGraph1["Runtime Function Calls"]
    PERCPU_AREA_BASE_CALL["percpu::percpu_area_base(cpu_id)"]
    SYMBOL_OFFSET_CALL["percpu_symbol_offset! macro"]
    READ_PERCPU_REG_CALL["read_percpu_reg()"]
end
subgraph subGraph0["Macro-Generated Code"]
    WRAPPER_METHODS["Wrapper Methods"]
    OFFSET_CALC["self.offset()"]
    CURRENT_PTR_CALC["self.current_ptr()"]
    REMOTE_PTR_CALC["self.remote_ptr(cpu_id)"]
    ASSEMBLY_ACCESS["Architecture-specific assembly"]
end

ASSEMBLY_ACCESS --> CPU_REGISTERS
ASSEMBLY_ACCESS --> READ_PERCPU_REG_CALL
CURRENT_PTR_CALC --> ASSEMBLY_ACCESS
INNER_SYMBOLS --> PERCPU_SECTION
LINKER_SYMBOLS --> PERCPU_SECTION
OFFSET_CALC --> SYMBOL_OFFSET_CALL
PERCPU_AREA_BASE_CALL --> CPU0_AREA
PERCPU_AREA_BASE_CALL --> CPU1_AREA
PERCPU_AREA_BASE_CALL --> CPUN_AREA
PERCPU_AREA_BASE_CALL --> LINKER_SYMBOLS
PERCPU_SECTION --> TEMPLATE_AREA
REMOTE_PTR_CALC --> PERCPU_AREA_BASE_CALL
SYMBOL_OFFSET_CALL --> INNER_SYMBOLS
TEMPLATE_AREA --> CPU0_AREA
TEMPLATE_AREA --> CPU1_AREA
TEMPLATE_AREA --> CPUN_AREA
```

Sources: [percpu_macros/src/lib.rs(L216 - L221)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L216-L221) [percpu/src/imp.rs(L32 - L44)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L32-L44) [percpu_macros/src/lib.rs(L255 - L261)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L255-L261)