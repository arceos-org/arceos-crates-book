# Feature Flags Configuration

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md)
> * [percpu/Cargo.toml](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/Cargo.toml)
> * [percpu_macros/Cargo.toml](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/Cargo.toml)

This document explains how to configure the percpu crate's feature flags to optimize per-CPU data management for different system architectures and use cases. It covers the three main feature flags (`sp-naive`, `preempt`, `arm-el2`) and their interaction with dependencies and code generation.

For basic usage examples and integration steps, see [Basic Usage Examples](/arceos-org/percpu/2.2-basic-usage-examples). For detailed implementation specifics of each feature, see [Naive Implementation](/arceos-org/percpu/5.2-naive-implementation).

## Overview of Feature Flags

The percpu crate provides three optional features that adapt the per-CPU data system to different operational environments and architectural requirements.

### Feature Flag Architecture

```mermaid
flowchart TD
subgraph subGraph2["Dependencies Activated"]
    KERNEL_GUARD["kernel_guard cratePreemption control"]
    MACRO_FLAGS["percpu_macros featuresCode generation control"]
end
subgraph subGraph1["Generated Code Variants"]
    MULTI_CPU["Multi-CPU ImplementationArchitecture-specific registers"]
    GLOBAL_VAR["Global Variable ImplementationNo per-CPU isolation"]
    GUARDED["Preemption-Safe ImplementationNoPreemptGuard wrapper"]
    EL2_IMPL["EL2 Register ImplementationTPIDR_EL2 usage"]
end
subgraph subGraph0["Feature Configuration"]
    DEFAULT["default = []Standard multi-CPU mode"]
    SP_NAIVE["sp-naiveSingle-core fallback"]
    PREEMPT["preemptPreemption safety"]
    ARM_EL2["arm-el2Hypervisor mode"]
end

ARM_EL2 --> EL2_IMPL
ARM_EL2 --> MACRO_FLAGS
DEFAULT --> MULTI_CPU
PREEMPT --> GUARDED
PREEMPT --> KERNEL_GUARD
PREEMPT --> MACRO_FLAGS
SP_NAIVE --> GLOBAL_VAR
SP_NAIVE --> MACRO_FLAGS
```

**Sources:** [percpu/Cargo.toml(L15 - L25)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/Cargo.toml#L15-L25) [percpu_macros/Cargo.toml(L15 - L25)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/Cargo.toml#L15-L25) [README.md(L69 - L79)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L69-L79)

## Feature Flag Descriptions

### sp-naive Feature

The `sp-naive` feature transforms per-CPU variables into standard global variables, eliminating per-CPU isolation for single-core systems.

|Aspect|Standard Mode|sp-naive Mode|
| --- | --- | --- |
|Variable Storage|Per-CPU memory areas|Global variables|
|Register Usage|Architecture-specific (GS_BASE, TPIDR_EL1, etc.)|None|
|Memory Layout|.percpu section with CPU multiplication|Standard .data/.bss sections|
|Access Method|CPU register + offset calculation|Direct global access|
|Initialization|percpu::init()andpercpu::init_percpu_reg()|Standard global initialization|

**Configuration:**

```
[dependencies]
percpu = { version = "0.2", features = ["sp-naive"] }
```

**Sources:** [percpu/Cargo.toml(L18 - L19)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/Cargo.toml#L18-L19) [percpu_macros/Cargo.toml(L18 - L19)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/Cargo.toml#L18-L19) [README.md(L71 - L73)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L71-L73)

### preempt Feature

The `preempt` feature enables preemption safety by integrating with the `kernel_guard` crate to prevent data corruption during per-CPU access.

### Feature Dependencies Chain

```mermaid
flowchart TD
subgraph subGraph3["Generated Access Methods"]
    READ_CURRENT["read_current()Preemption-safe read"]
    WRITE_CURRENT["write_current()Preemption-safe write"]
    NO_PREEMPT_GUARD["NoPreemptGuardRAII guard"]
end
subgraph subGraph2["percpu_macros Crate"]
    MACROS_PREEMPT["preempt featurepercpu_macros/Cargo.toml:22"]
    CODE_GEN["Code GenerationNoPreemptGuard integration"]
end
subgraph subGraph1["percpu Crate"]
    PERCPU_PREEMPT["preempt featurepercpu/Cargo.toml:22"]
    KERNEL_GUARD_DEP["kernel_guard dependencyConditionally enabled"]
end
subgraph subGraph0["User Configuration"]
    USER_TOML["Cargo.tomlfeatures = ['preempt']"]
end

CODE_GEN --> NO_PREEMPT_GUARD
CODE_GEN --> READ_CURRENT
CODE_GEN --> WRITE_CURRENT
KERNEL_GUARD_DEP --> NO_PREEMPT_GUARD
MACROS_PREEMPT --> CODE_GEN
PERCPU_PREEMPT --> KERNEL_GUARD_DEP
PERCPU_PREEMPT --> MACROS_PREEMPT
USER_TOML --> PERCPU_PREEMPT
```

**Configuration:**

```
[dependencies]
percpu = { version = "0.2", features = ["preempt"] }
```

**Sources:** [percpu/Cargo.toml(L21 - L22)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/Cargo.toml#L21-L22) [percpu_macros/Cargo.toml(L21 - L22)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/Cargo.toml#L21-L22) [README.md(L74 - L76)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L74-L76)

### arm-el2 Feature

The `arm-el2` feature configures AArch64 systems to use the EL2 (Exception Level 2) thread pointer register for hypervisor environments.

|Register Usage|Default (arm-el2 disabled)|arm-el2 enabled|
| --- | --- | --- |
|AArch64 Register|TPIDR_EL1|TPIDR_EL2|
|Privilege Level|EL1 (kernel mode)|EL2 (hypervisor mode)|
|Use Case|Standard kernel|Hypervisor/VMM|
|Assembly Instructions|mrs/msrwithTPIDR_EL1|mrs/msrwithTPIDR_EL2|

**Configuration:**

```
[dependencies]
percpu = { version = "0.2", features = ["arm-el2"] }
```

**Sources:** [percpu/Cargo.toml(L24 - L25)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/Cargo.toml#L24-L25) [percpu_macros/Cargo.toml(L24 - L25)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/Cargo.toml#L24-L25) [README.md(L33 - L35)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L33-L35) [README.md(L77 - L79)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L77-L79)

## Feature Combination Matrix

### Compatible Feature Combinations

```mermaid
flowchart TD
subgraph subGraph1["Architecture Applicability"]
    X86_COMPAT["x86_64 CompatibleGS_BASE register"]
    ARM_COMPAT["AArch64 CompatibleTPIDR_ELx registers"]
    RISCV_COMPAT["RISC-V Compatiblegp register"]
    LOONG_COMPAT["LoongArch Compatible$r21 register"]
end
subgraph subGraph0["Valid Configurations"]
    DEFAULT_ONLY["DefaultMulti-CPU, no preemption"]
    SP_NAIVE_ONLY["sp-naiveSingle-core system"]
    PREEMPT_ONLY["preemptMulti-CPU with preemption"]
    ARM_EL2_ONLY["arm-el2AArch64 hypervisor"]
    PREEMPT_ARM["preempt + arm-el2Hypervisor with preemption"]
    SP_NAIVE_PREEMPT["sp-naive + preemptSingle-core with preemption"]
    SP_NAIVE_ARM["sp-naive + arm-el2Single-core hypervisor"]
    ALL_FEATURES["sp-naive + preempt + arm-el2Single-core hypervisor with preemption"]
end

ARM_EL2_ONLY --> ARM_COMPAT
DEFAULT_ONLY --> ARM_COMPAT
DEFAULT_ONLY --> LOONG_COMPAT
DEFAULT_ONLY --> RISCV_COMPAT
DEFAULT_ONLY --> X86_COMPAT
PREEMPT_ARM --> ARM_COMPAT
PREEMPT_ONLY --> ARM_COMPAT
PREEMPT_ONLY --> LOONG_COMPAT
PREEMPT_ONLY --> RISCV_COMPAT
PREEMPT_ONLY --> X86_COMPAT
SP_NAIVE_ONLY --> ARM_COMPAT
SP_NAIVE_ONLY --> LOONG_COMPAT
SP_NAIVE_ONLY --> RISCV_COMPAT
SP_NAIVE_ONLY --> X86_COMPAT
```

**Sources:** [percpu/Cargo.toml(L15 - L25)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/Cargo.toml#L15-L25) [percpu_macros/Cargo.toml(L15 - L25)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/Cargo.toml#L15-L25)

## Configuration Examples

### Bare Metal Single-Core System

For embedded systems or single-core environments:

```
[dependencies]
percpu = { version = "0.2", features = ["sp-naive"] }
```

This configuration eliminates per-CPU overhead and uses simple global variables.

### Multi-Core Preemptible Kernel

For operating system kernels with preemptive scheduling:

```
[dependencies]
percpu = { version = "0.2", features = ["preempt"] }
```

This enables preemption safety with `NoPreemptGuard` wrappers around access operations.

### AArch64 Hypervisor

For hypervisors running at EL2 privilege level:

```
[dependencies]
percpu = { version = "0.2", features = ["arm-el2"] }
```

This uses `TPIDR_EL2` instead of `TPIDR_EL1` for per-CPU base address storage.

### Multi-Core Hypervisor with Preemption

For complex hypervisor environments:

```
[dependencies]
percpu = { version = "0.2", features = ["preempt", "arm-el2"] }
```

This combines EL2 register usage with preemption safety mechanisms.

## Conditional Compilation Impact

### Feature-Dependent Code Paths

```mermaid
flowchart TD
subgraph subGraph2["Generated Code Variants"]
    GLOBAL_IMPL["Global Variable ImplementationDirect access"]
    PERCPU_IMPL["Per-CPU ImplementationRegister-based access"]
    GUARDED_IMPL["Guarded ImplementationNoPreemptGuard wrapper"]
    EL2_IMPL["EL2 ImplementationTPIDR_EL2 usage"]
end
subgraph subGraph1["Feature Detection"]
    CFG_SP_NAIVE["#[cfg(feature = 'sp-naive')]"]
    CFG_PREEMPT["#[cfg(feature = 'preempt')]"]
    CFG_ARM_EL2["#[cfg(feature = 'arm-el2')]"]
end
subgraph subGraph0["Macro Expansion"]
    DEF_PERCPU["#[def_percpu]static VAR: T = init;"]
end

CFG_ARM_EL2 --> EL2_IMPL
CFG_PREEMPT --> GUARDED_IMPL
CFG_SP_NAIVE --> GLOBAL_IMPL
CFG_SP_NAIVE --> PERCPU_IMPL
DEF_PERCPU --> CFG_ARM_EL2
DEF_PERCPU --> CFG_PREEMPT
DEF_PERCPU --> CFG_SP_NAIVE
PERCPU_IMPL --> EL2_IMPL
PERCPU_IMPL --> GUARDED_IMPL
```

**Sources:** [percpu_macros/Cargo.toml(L15 - L25)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/Cargo.toml#L15-L25)

## Architecture-Specific Considerations

### x86_64 Systems

All feature combinations are supported. The `arm-el2` feature has no effect on x86_64 targets.

**Register Used:** `GS_BASE` (unless `sp-naive` is enabled)

### AArch64 Systems

The `arm-el2` feature specifically affects AArch64 compilation:

* **Default:** Uses `TPIDR_EL1` register
* **With arm-el2:** Uses `TPIDR_EL2` register

### RISC-V Systems

The `arm-el2` feature has no effect. Uses `gp` register for per-CPU base address.

### LoongArch Systems

The `arm-el2` feature has no effect. Uses `$r21` register for per-CPU base address.

**Sources:** [README.md(L19 - L36)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L19-L36)

## Dependencies and Build Impact

### Conditional Dependencies

The feature flags control which dependencies are included in the build:

|Feature|Additional Dependencies|Purpose|
| --- | --- | --- |
|sp-naive|None|Simplifies to global variables|
|preempt|kernel_guard = "0.1"|ProvidesNoPreemptGuard|
|arm-el2|None|Only affects macro code generation|

### Build Optimization

When `sp-naive` is enabled, the generated code is significantly simpler and may result in better optimization for single-core systems since it eliminates the per-CPU indirection overhead.

**Sources:** [percpu/Cargo.toml(L27 - L36)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/Cargo.toml#L27-L36)