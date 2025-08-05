# API Reference

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md)
> * [percpu/src/lib.rs](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/lib.rs)
> * [percpu_macros/src/lib.rs](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs)

This document provides a comprehensive reference for all public APIs, macros, and functions in the percpu crate ecosystem. It covers the main user-facing interfaces for defining and accessing per-CPU data structures.

For detailed information about the `def_percpu` macro syntax and usage patterns, see [def_percpu Macro](/arceos-org/percpu/4.1-def_percpu-macro). For runtime initialization and management functions, see [Runtime Functions](/arceos-org/percpu/4.2-runtime-functions). For guidance on safe usage and preemption handling, see [Safety and Preemption](/arceos-org/percpu/4.3-safety-and-preemption).

## API Overview

The percpu crate provides a declarative interface for per-CPU data management through the `def_percpu` attribute macro and a set of runtime functions for system initialization.

### Complete API Surface

```mermaid
flowchart TD
subgraph subGraph5["Internal APIs"]
    PERCPU_AREA_BASE["percpu_area_base(cpu_id) -> usize"]
    NOPREEMPT_GUARD["NoPreemptGuard"]
    PERCPU_SYMBOL_OFFSET["percpu_symbol_offset!"]
end
subgraph subGraph4["Generated Per-Variable APIs"]
    WRAPPER["VAR_WRAPPERZero-sized struct"]
    STATIC_VAR["VAR: VAR_WRAPPERStatic instance"]
    subgraph subGraph3["Primitive Type Methods"]
        READ_CURRENT_RAW["read_current_raw() -> T"]
        WRITE_CURRENT_RAW["write_current_raw(val: T)"]
        READ_CURRENT["read_current() -> T"]
        WRITE_CURRENT["write_current(val: T)"]
    end
    subgraph subGraph2["Raw Access Methods"]
        CURRENT_REF_RAW["current_ref_raw() -> &T"]
        CURRENT_REF_MUT_RAW["current_ref_mut_raw() -> &mut T"]
        REMOTE_PTR["remote_ptr(cpu_id) -> *const T"]
        REMOTE_REF_RAW["remote_ref_raw(cpu_id) -> &T"]
        REMOTE_REF_MUT_RAW["remote_ref_mut_raw(cpu_id) -> &mut T"]
    end
    subgraph subGraph1["Core Methods"]
        OFFSET["offset() -> usize"]
        CURRENT_PTR["current_ptr() -> *const T"]
        WITH_CURRENT["with_current(f: F) -> R"]
    end
end
subgraph subGraph0["User Interface"]
    DEF_PERCPU["#[def_percpu]Attribute Macro"]
    INIT["percpu::init()"]
    INIT_REG["percpu::init_percpu_reg(cpu_id)"]
end

DEF_PERCPU --> STATIC_VAR
DEF_PERCPU --> WRAPPER
READ_CURRENT --> NOPREEMPT_GUARD
REMOTE_PTR --> PERCPU_AREA_BASE
WITH_CURRENT --> NOPREEMPT_GUARD
WRAPPER --> CURRENT_PTR
WRAPPER --> CURRENT_REF_MUT_RAW
WRAPPER --> CURRENT_REF_RAW
WRAPPER --> OFFSET
WRAPPER --> READ_CURRENT
WRAPPER --> READ_CURRENT_RAW
WRAPPER --> REMOTE_PTR
WRAPPER --> REMOTE_REF_MUT_RAW
WRAPPER --> REMOTE_REF_RAW
WRAPPER --> WITH_CURRENT
WRAPPER --> WRITE_CURRENT
WRAPPER --> WRITE_CURRENT_RAW
WRITE_CURRENT --> NOPREEMPT_GUARD
```

**Sources:** [percpu_macros/src/lib.rs(L66 - L252)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L66-L252) [percpu/src/lib.rs(L5 - L17)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/lib.rs#L5-L17) [README.md(L39 - L52)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L39-L52)

## Core API Components

### Macro Definition Interface

The primary user interface is the `def_percpu` attribute macro that transforms static variable definitions into per-CPU data structures.

|Component|Type|Purpose|
| --- | --- | --- |
|#[def_percpu]|Attribute Macro|Transforms static variables into per-CPU data|
|percpu::init()|Function|Initializes per-CPU data areas|
|percpu::init_percpu_reg(cpu_id)|Function|Sets up per-CPU register for given CPU|

**Sources:** [percpu_macros/src/lib.rs(L66 - L78)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L66-L78) [percpu/src/lib.rs(L11)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/lib.rs#L11-L11) [README.md(L44 - L46)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L44-L46)

### Generated Code Structure

For each variable defined with `#[def_percpu]`, the macro generates multiple code entities that work together to provide per-CPU access.

```mermaid
flowchart TD
subgraph subGraph3["Method Categories"]
    OFFSET_METHODS["Offset Calculationoffset()"]
    POINTER_METHODS["Pointer Accesscurrent_ptr(), remote_ptr()"]
    REFERENCE_METHODS["Reference Accesscurrent_ref_raw(), remote_ref_raw()"]
    VALUE_METHODS["Value Accessread_current(), write_current()"]
    FUNCTIONAL_METHODS["Functional Accesswith_current()"]
end
subgraph subGraph2["Section Placement"]
    PERCPU_SECTION[".percpu sectionLinker managed"]
end
subgraph subGraph1["Generated Entities"]
    INNER_SYMBOL["__PERCPU_VARstatic mut __PERCPU_VAR: T"]
    WRAPPER_STRUCT["VAR_WRAPPERstruct VAR_WRAPPER {}"]
    PUBLIC_STATIC["VARstatic VAR: VAR_WRAPPER"]
end
subgraph subGraph0["User Declaration"]
    USER_VAR["#[def_percpu]static VAR: T = init;"]
end

INNER_SYMBOL --> PERCPU_SECTION
USER_VAR --> INNER_SYMBOL
USER_VAR --> PUBLIC_STATIC
USER_VAR --> WRAPPER_STRUCT
WRAPPER_STRUCT --> FUNCTIONAL_METHODS
WRAPPER_STRUCT --> OFFSET_METHODS
WRAPPER_STRUCT --> POINTER_METHODS
WRAPPER_STRUCT --> REFERENCE_METHODS
WRAPPER_STRUCT --> VALUE_METHODS
```

**Sources:** [percpu_macros/src/lib.rs(L88 - L159)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L88-L159) [percpu_macros/src/lib.rs(L33 - L51)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L33-L51)

## Method Categories and Safety

The generated wrapper struct provides methods organized into distinct categories based on their safety requirements and access patterns.

### Method Safety Hierarchy

```mermaid
flowchart TD
subgraph subGraph4["Preemption Guard Integration"]
    NOPREEMPT["NoPreemptGuardkernel_guard crate"]
end
subgraph subGraph3["Utility Methods (Always Safe)"]
    OFFSET_SAFE["offset() -> usizeAll types"]
end
subgraph subGraph2["Remote Access (Always Unsafe)"]
    REMOTE_PTR_UNSAFE["remote_ptr(cpu_id) -> *const TAll types"]
    REMOTE_REF_RAW_UNSAFE["remote_ref_raw(cpu_id) -> &TAll types"]
    REMOTE_REF_MUT_RAW_UNSAFE["remote_ref_mut_raw(cpu_id) -> &mut TAll types"]
end
subgraph subGraph1["Unsafe Methods (Manual Preemption Control)"]
    READ_CURRENT_RAW_UNSAFE["read_current_raw() -> TPrimitive types only"]
    WRITE_CURRENT_RAW_UNSAFE["write_current_raw(val: T)Primitive types only"]
    CURRENT_REF_RAW_UNSAFE["current_ref_raw() -> &TAll types"]
    CURRENT_REF_MUT_RAW_UNSAFE["current_ref_mut_raw() -> &mut TAll types"]
    CURRENT_PTR_UNSAFE["current_ptr() -> *const TAll types"]
end
subgraph subGraph0["Safe Methods (Preemption Handled)"]
    READ_CURRENT_SAFE["read_current() -> TPrimitive types only"]
    WRITE_CURRENT_SAFE["write_current(val: T)Primitive types only"]
    WITH_CURRENT_SAFE["with_current(f: F) -> RAll types"]
end

READ_CURRENT_SAFE --> NOPREEMPT
READ_CURRENT_SAFE --> READ_CURRENT_RAW_UNSAFE
REMOTE_PTR_UNSAFE --> OFFSET_SAFE
REMOTE_REF_MUT_RAW_UNSAFE --> REMOTE_PTR_UNSAFE
REMOTE_REF_RAW_UNSAFE --> REMOTE_PTR_UNSAFE
WITH_CURRENT_SAFE --> CURRENT_REF_MUT_RAW_UNSAFE
WITH_CURRENT_SAFE --> NOPREEMPT
WRITE_CURRENT_SAFE --> NOPREEMPT
WRITE_CURRENT_SAFE --> WRITE_CURRENT_RAW_UNSAFE
```

**Sources:** [percpu_macros/src/lib.rs(L101 - L145)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L101-L145) [percpu_macros/src/lib.rs(L161 - L248)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L161-L248) [percpu/src/lib.rs(L14 - L17)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/lib.rs#L14-L17)

## Type-Specific API Variations

The generated API surface varies depending on the type of the per-CPU variable, with additional optimized methods for primitive integer types.

### API Method Availability Matrix

|Method Category|All Types|Primitive Integer Types Only|
| --- | --- | --- |
|offset()|✓|✓|
|current_ptr()|✓|✓|
|current_ref_raw()|✓|✓|
|current_ref_mut_raw()|✓|✓|
|with_current()|✓|✓|
|remote_ptr()|✓|✓|
|remote_ref_raw()|✓|✓|
|remote_ref_mut_raw()|✓|✓|
|read_current_raw()|✗|✓|
|write_current_raw()|✗|✓|
|read_current()|✗|✓|
|write_current()|✗|✓|

**Primitive integer types:** `bool`, `u8`, `u16`, `u32`, `u64`, `usize`

**Sources:** [percpu_macros/src/lib.rs(L91 - L93)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L91-L93) [percpu_macros/src/lib.rs(L100 - L145)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L100-L145)

## Feature-Dependent Behavior

The API behavior changes based on enabled Cargo features, affecting both code generation and runtime behavior.

### Feature Impact on API

|Feature|Impact on Generated Code|Runtime Behavior|
| --- | --- | --- |
|sp-naive|Uses global variables instead of per-CPU areas|No register setup required|
|preempt|AddsNoPreemptGuardto safe methods|Preemption disabled during access|
|arm-el2|Changes register selection for AArch64|UsesTPIDR_EL2instead ofTPIDR_EL1|

**Sources:** [percpu_macros/src/lib.rs(L94 - L98)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L94-L98) [percpu/src/lib.rs(L7 - L8)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/lib.rs#L7-L8) [README.md(L69 - L79)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/README.md#L69-L79)