# Supported Platforms

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/percpu/blob/89c8a54c/.github/workflows/ci.yml)
> * [README.md](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md)
> * [percpu_macros/src/arch.rs](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs)

This document provides a comprehensive overview of the CPU architectures and platforms supported by the percpu crate ecosystem. It covers architecture-specific per-CPU register usage, implementation details, and platform-specific considerations for integrating per-CPU data management.

For information about setting up these platforms in your build environment, see [Installation and Setup](/arceos-org/percpu/2.1-installation-and-setup). For details about the architecture-specific code generation mechanisms, see [Architecture-Specific Code Generation](/arceos-org/percpu/5.1-architecture-specific-code-generation).

## Supported Architecture Overview

The percpu crate supports four major CPU architectures, each utilizing different registers for per-CPU data access:

|Architecture|per-CPU Register|Register Type|Supported Variants|
| --- | --- | --- | --- |
|x86_64|GS_BASE|Segment base register|Standard x86_64|
|AArch64|TPIDR_ELx|Thread pointer register|EL1, EL2 modes|
|RISC-V|gp|Global pointer register|32-bit, 64-bit|
|LoongArch|$r21|General purpose register|64-bit|

**Platform Register Usage**

```mermaid
flowchart TD
subgraph subGraph2["Access Methods"]
    X86_ASM["mov gs:[offset]"]
    ARM_ASM["mrs/msr instructions"]
    RISCV_ASM["lui/addi with gp"]
    LOONG_ASM["lu12i.w/ori with $r21"]
end
subgraph subGraph1["Per-CPU Registers"]
    GS["GS_BASESegment Register"]
    TPIDR["TPIDR_ELxThread Pointer"]
    GP["gpGlobal Pointer"]
    R21["$r21General Purpose"]
end
subgraph subGraph0["Architecture Support"]
    X86["x86_64"]
    ARM["AArch64"]
    RISCV["RISC-V"]
    LOONG["LoongArch"]
end

ARM --> TPIDR
GP --> RISCV_ASM
GS --> X86_ASM
LOONG --> R21
R21 --> LOONG_ASM
RISCV --> GP
TPIDR --> ARM_ASM
X86 --> GS
```

Sources: [README.md(L19 - L31)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L19-L31) [percpu_macros/src/arch.rs(L21 - L46)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L21-L46)

## Platform-Specific Implementation Details

### x86_64 Architecture

The x86_64 implementation uses the `GS_BASE` model-specific register to store the base address of the per-CPU data area. This approach leverages the x86_64 segment architecture for efficient per-CPU data access.

**Key characteristics:**

* Uses `GS_BASE` MSR for per-CPU base address storage
* Accesses data via `gs:[offset]` addressing mode
* Requires offset values ≤ 0xffff_ffff for 32-bit displacement
* Supports optimized single-instruction memory operations

```mermaid
flowchart TD
subgraph subGraph1["Generated Assembly"]
    MOV_READ["mov reg, gs:[offset VAR]Direct read"]
    MOV_WRITE["mov gs:[offset VAR], regDirect write"]
    ADDR_CALC["mov reg, offset VARadd reg, gs:[__PERCPU_SELF_PTR]"]
end
subgraph subGraph0["x86_64 Implementation"]
    GS_BASE["GS_BASE MSRBase Address"]
    OFFSET["Symbol OffsetCalculated at compile-time"]
    ACCESS["gs:[offset] AccessSingle instruction"]
end

ACCESS --> ADDR_CALC
ACCESS --> MOV_READ
ACCESS --> MOV_WRITE
GS_BASE --> ACCESS
OFFSET --> ACCESS
```

Sources: [percpu_macros/src/arch.rs(L66 - L76)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L66-L76) [percpu_macros/src/arch.rs(L131 - L150)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L131-L150) [percpu_macros/src/arch.rs(L232 - L251)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L232-L251)

### AArch64 Architecture

The AArch64 implementation uses thread pointer registers that vary based on the exception level. The system supports both EL1 (kernel) and EL2 (hypervisor) execution environments.

**Key characteristics:**

* Uses `TPIDR_EL1` by default, `TPIDR_EL2` when `arm-el2` feature is enabled
* Requires `mrs`/`msr` instructions for register access
* Offset calculations limited to 16-bit immediate values (≤ 0xffff)
* Two-instruction sequence for per-CPU data access

The register selection is controlled at compile time:

```mermaid
flowchart TD
subgraph subGraph2["Access Pattern"]
    MRS["mrs reg, TPIDR_ELxGet base address"]
    CALC["add reg, offsetCalculate final address"]
    ACCESS["ldr/str operationsMemory access"]
end
subgraph subGraph1["Register Usage"]
    EL1["TPIDR_EL1Kernel Mode"]
    EL2["TPIDR_EL2Hypervisor Mode"]
end
subgraph subGraph0["AArch64 Register Selection"]
    FEATURE["arm-el2 feature"]
    DEFAULT["Default Mode"]
end

CALC --> ACCESS
DEFAULT --> EL1
EL1 --> MRS
EL2 --> MRS
FEATURE --> EL2
MRS --> CALC
```

Sources: [percpu_macros/src/arch.rs(L55 - L62)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L55-L62) [percpu_macros/src/arch.rs(L79 - L80)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L79-L80) [README.md(L33 - L35)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L33-L35)

### RISC-V Architecture

The RISC-V implementation uses the global pointer (`gp`) register for per-CPU data base addressing. This is a deviation from typical RISC-V conventions where `gp` is used for global data access.

**Key characteristics:**

* Uses `gp` register instead of standard thread pointer (`tp`)
* Supports both 32-bit and 64-bit variants
* Uses `lui`/`addi` instruction sequence for address calculation
* Offset values limited to 32-bit signed immediate range

**Important note:** The `tp` register remains available for thread-local storage, while `gp` is repurposed for per-CPU data.

```mermaid
flowchart TD
subgraph subGraph2["Memory Operations"]
    LOAD["ld/lw/lh/lb instructionsBased on data type"]
    STORE["sd/sw/sh/sb instructionsBased on data type"]
end
subgraph subGraph1["Address Calculation"]
    LUI["lui reg, %hi(VAR)Load upper immediate"]
    ADDI["addi reg, reg, %lo(VAR)Add lower immediate"]
    FINAL["add reg, reg, gpAdd base address"]
end
subgraph subGraph0["RISC-V Register Usage"]
    GP["gp registerPer-CPU base"]
    TP["tp registerThread-local storage"]
    OFFSET["Symbol offset%hi/%lo split"]
end

ADDI --> FINAL
FINAL --> LOAD
FINAL --> STORE
GP --> FINAL
LUI --> ADDI
OFFSET --> LUI
```

Sources: [percpu_macros/src/arch.rs(L81 - L82)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L81-L82) [percpu_macros/src/arch.rs(L33 - L39)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L33-L39) [README.md(L28 - L31)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L28-L31)

### LoongArch Architecture

The LoongArch implementation uses the `$r21` general-purpose register for per-CPU data base addressing. This architecture provides native support for per-CPU data patterns.

**Key characteristics:**

* Uses `$r21` general-purpose register for base addressing
* Supports 64-bit architecture (LoongArch64)
* Uses `lu12i.w`/`ori` instruction sequence for address calculation
* Provides specialized load/store indexed instructions

```mermaid
flowchart TD
subgraph subGraph1["Instruction Sequences"]
    LU12I["lu12i.w reg, %abs_hi20(VAR)Load upper 20 bits"]
    ORI["ori reg, reg, %abs_lo12(VAR)OR lower 12 bits"]
    LDX["ldx.d/ldx.w/ldx.h/ldx.buLoad indexed"]
    STX["stx.d/stx.w/stx.h/stx.bStore indexed"]
end
subgraph subGraph0["LoongArch Implementation"]
    R21["$r21 registerPer-CPU base"]
    CALC["Address calculationlu12i.w + ori"]
    INDEXED["Indexed operationsldx/stx instructions"]
end

CALC --> LU12I
LU12I --> ORI
ORI --> LDX
ORI --> STX
R21 --> INDEXED
```

Sources: [percpu_macros/src/arch.rs(L83 - L84)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L83-L84) [percpu_macros/src/arch.rs(L40 - L46)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L40-L46) [percpu_macros/src/arch.rs(L114 - L129)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L114-L129)

## Continuous Integration and Testing Coverage

The percpu crate maintains comprehensive testing across all supported platforms through automated CI/CD pipelines. The testing matrix ensures compatibility and correctness across different target environments.

**CI Target Matrix:**

|Target|Architecture|Environment|Test Coverage|
| --- | --- | --- | --- |
|x86_64-unknown-linux-gnu|x86_64|Linux userspace|Full tests + unit tests|
|x86_64-unknown-none|x86_64|Bare metal/no_std|Build + clippy only|
|riscv64gc-unknown-none-elf|RISC-V 64|Bare metal/no_std|Build + clippy only|
|aarch64-unknown-none-softfloat|AArch64|Bare metal/no_std|Build + clippy only|
|loongarch64-unknown-none-softfloat|LoongArch64|Bare metal/no_std|Build + clippy only|

**Testing Strategy:**

```mermaid
flowchart TD
subgraph subGraph2["Feature Combinations"]
    SP_NAIVE["sp-naive featureSingle-core testing"]
    PREEMPT["preempt featurePreemption safety"]
    ARM_EL2["arm-el2 featureHypervisor mode"]
end
subgraph subGraph1["Test Types"]
    BUILD["Build TestsAll targets"]
    UNIT["Unit Testsx86_64 Linux only"]
    LINT["Code QualityAll targets"]
end
subgraph subGraph0["CI Pipeline"]
    MATRIX["Target Matrix5 platforms tested"]
    FEATURES["Feature Testingpreempt, arm-el2"]
    TOOLS["Quality Toolsclippy, rustfmt"]
end

BUILD --> ARM_EL2
BUILD --> PREEMPT
BUILD --> SP_NAIVE
FEATURES --> ARM_EL2
FEATURES --> PREEMPT
FEATURES --> SP_NAIVE
MATRIX --> BUILD
MATRIX --> LINT
MATRIX --> UNIT
```

Sources: [.github/workflows/ci.yml(L8 - L32)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/.github/workflows/ci.yml#L8-L32)

## Platform-Specific Limitations and Considerations

### macOS Development Limitation

The crate includes a development-time limitation for macOS hosts, where architecture-specific assembly code is disabled and replaced with unimplemented stubs. This affects local development but not target deployment.

### Offset Size Constraints

Each architecture imposes different constraints on the maximum offset values for per-CPU variables:

* **x86_64**: 32-bit signed displacement (≤ 0xffff_ffff)
* **AArch64**: 16-bit immediate value (≤ 0xffff)
* **RISC-V**: 32-bit signed immediate (split hi/lo)
* **LoongArch**: 32-bit absolute address (split hi20/lo12)

### Register Conflicts and Conventions

* **RISC-V**: Repurposes `gp` register, deviating from standard ABI conventions
* **AArch64**: Requires careful exception level management for register selection
* **x86_64**: Depends on GS segment register configuration by system initialization

Sources: [percpu_macros/src/arch.rs(L4 - L13)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L4-L13) [percpu_macros/src/arch.rs(L23 - L46)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L23-L46) [README.md(L28 - L35)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L28-L35)