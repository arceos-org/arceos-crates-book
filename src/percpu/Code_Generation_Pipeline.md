# Code Generation Pipeline

> **Relevant source files**
> * [percpu_macros/Cargo.toml](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/Cargo.toml)
> * [percpu_macros/src/arch.rs](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs)
> * [percpu_macros/src/lib.rs](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs)

This document details the compile-time transformation pipeline that converts user-defined per-CPU variable declarations into architecture-specific, optimized access code. The pipeline is implemented as a procedural macro system that analyzes user code, detects target platform characteristics, and generates efficient per-CPU data access methods.

For information about the runtime memory layout and initialization, see [Memory Layout and Initialization](/arceos-org/percpu/3.1-memory-layout-and-initialization). For details about the specific assembly code generated for each architecture, see [Architecture-Specific Code Generation](/arceos-org/percpu/5.1-architecture-specific-code-generation).

## Pipeline Overview

The code generation pipeline transforms a simple user declaration into a comprehensive per-CPU data access system through multiple stages of analysis and code generation.

### High-Level Transformation Flow

```mermaid
flowchart TD
subgraph subGraph2["Output Components"]
    OUTPUT["Generated Components"]
    STORAGE["__PERCPU_VAR in .percpu section"]
    WRAPPER["VAR_WRAPPER struct"]
    INTERFACE["VAR: VAR_WRAPPER static"]
end
subgraph subGraph1["Generation Phase"]
    GENERATE["Code Generation"]
    STORAGE_GEN["__PERCPU_VAR generation"]
    WRAPPER_GEN["VAR_WRAPPER generation"]
    METHOD_GEN["Method generation"]
    ARCH_GEN["Architecture-specific code"]
end
subgraph subGraph0["Analysis Phase"]
    ANALYZE["Type & Feature Analysis"]
    TYPE_CHECK["is_primitive_int check"]
    FEATURE_CHECK["Feature flag detection"]
    SYMBOL_GEN["Symbol name generation"]
end
INPUT["#[def_percpu] static VAR: T = init;"]
PARSE["syn::parse_macro_input"]

ANALYZE --> FEATURE_CHECK
ANALYZE --> GENERATE
ANALYZE --> SYMBOL_GEN
ANALYZE --> TYPE_CHECK
GENERATE --> ARCH_GEN
GENERATE --> METHOD_GEN
GENERATE --> OUTPUT
GENERATE --> STORAGE_GEN
GENERATE --> WRAPPER_GEN
INPUT --> PARSE
OUTPUT --> INTERFACE
OUTPUT --> STORAGE
OUTPUT --> WRAPPER
PARSE --> ANALYZE
```

**Sources:** [percpu_macros/src/lib.rs(L72 - L252)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L72-L252)

## Input Parsing and Analysis

The pipeline begins with the `def_percpu` procedural macro, which uses the `syn` crate to parse the user's static variable declaration into an Abstract Syntax Tree (AST).

### Parsing Stage

```mermaid
flowchart TD
subgraph subGraph0["Extracted Components"]
    EXTRACT["Component Extraction"]
    ATTRS["attrs: &[Attribute]"]
    VIS["vis: &Visibility"]
    NAME["ident: &Ident"]
    TYPE["ty: &Type"]
    INIT["expr: &Expr"]
end
USER["#[def_percpu]static COUNTER: u64 = 0;"]
AST["ItemStatic AST"]

AST --> EXTRACT
EXTRACT --> ATTRS
EXTRACT --> INIT
EXTRACT --> NAME
EXTRACT --> TYPE
EXTRACT --> VIS
USER --> AST
```

The parsing logic extracts key components from the declaration:

|Component|Purpose|Example|
| --- | --- | --- |
|attrs|Preserve original attributes|#[no_mangle]|
|vis|Maintain visibility|pub|
|ident|Variable name|COUNTER|
|ty|Type information|u64|
|expr|Initialization expression|0|

**Sources:** [percpu_macros/src/lib.rs(L80 - L86)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L80-L86)

### Type Analysis and Symbol Generation

The pipeline performs type analysis to determine code generation strategy and generates internal symbol names:

```mermaid
flowchart TD
subgraph subGraph1["Generated Symbols"]
    SYMBOL_GEN["Symbol Name Generation"]
    INNER["_PERCPU{name}"]
    WRAPPER["{name}_WRAPPER"]
    PUBLIC["{name}"]
end
subgraph subGraph0["Primitive Type Detection"]
    PRIMITIVE_CHECK["is_primitive_int check"]
    BOOL["bool"]
    U8["u8"]
    U16["u16"]
    U32["u32"]
    U64["u64"]
    USIZE["usize"]
end
TYPE_ANALYSIS["Type Analysis"]

PRIMITIVE_CHECK --> BOOL
PRIMITIVE_CHECK --> U16
PRIMITIVE_CHECK --> U32
PRIMITIVE_CHECK --> U64
PRIMITIVE_CHECK --> U8
PRIMITIVE_CHECK --> USIZE
SYMBOL_GEN --> INNER
SYMBOL_GEN --> PUBLIC
SYMBOL_GEN --> WRAPPER
TYPE_ANALYSIS --> PRIMITIVE_CHECK
TYPE_ANALYSIS --> SYMBOL_GEN
```

The type analysis at [percpu_macros/src/lib.rs(L91 - L92)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L91-L92) determines whether to generate optimized assembly access methods for primitive integer types.

**Sources:** [percpu_macros/src/lib.rs(L88 - L92)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L88-L92)

## Code Generation Stages

The pipeline generates three primary components for each per-CPU variable, with different methods and optimizations based on type and feature analysis.

### Component Generation Structure

```mermaid
flowchart TD
subgraph subGraph2["Interface Component"]
    INTERFACE["VAR static variable"]
    PUBLIC_API["Public API access point"]
    ZERO_SIZED["Zero-sized type"]
end
subgraph subGraph1["Wrapper Component"]
    WRAPPER["VAR_WRAPPER struct"]
    BASIC_METHODS["Basic access methods"]
    ARCH_METHODS["Architecture-specific methods"]
    PRIMITIVE_METHODS["Primitive type methods"]
end
subgraph subGraph0["Storage Component"]
    STORAGE["__PERCPU_VAR"]
    SECTION[".percpu section placement"]
    ATTRS_PRESERVE["Preserve original attributes"]
    INIT_PRESERVE["Preserve initialization"]
end
GENERATION["Code Generation"]

GENERATION --> INTERFACE
GENERATION --> STORAGE
GENERATION --> WRAPPER
INTERFACE --> PUBLIC_API
INTERFACE --> ZERO_SIZED
STORAGE --> ATTRS_PRESERVE
STORAGE --> INIT_PRESERVE
STORAGE --> SECTION
WRAPPER --> ARCH_METHODS
WRAPPER --> BASIC_METHODS
WRAPPER --> PRIMITIVE_METHODS
```

**Sources:** [percpu_macros/src/lib.rs(L149 - L251)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L149-L251)

### Method Generation Logic

The wrapper struct receives different sets of methods based on type analysis and feature configuration:

|Method Category|Condition|Generated Methods|
| --- | --- | --- |
|Basic Access|Always|offset(),current_ptr(),current_ref_raw(),current_ref_mut_raw(),with_current(),remote_ptr(),remote_ref_raw(),remote_ref_mut_raw()|
|Primitive Optimized|is_primitive_int == true|read_current_raw(),write_current_raw(),read_current(),write_current()|
|Preemption Safety|feature = "preempt"|AutomaticNoPreemptGuardin safe methods|

**Sources:** [percpu_macros/src/lib.rs(L100 - L145)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L100-L145) [percpu_macros/src/lib.rs(L161 - L249)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L161-L249)

## Architecture-Specific Code Generation

The pipeline delegates architecture-specific code generation to specialized functions in the `arch` module, which produce optimized assembly code for each supported platform.

### Architecture Detection and Code Selection

```mermaid
flowchart TD
subgraph subGraph2["Register Access Patterns"]
    X86_GS["x86_64: gs:[offset __PERCPU_SELF_PTR]"]
    ARM_TPIDR["aarch64: mrs TPIDR_EL1/EL2"]
    RISCV_GP["riscv: mv gp"]
    LOONG_R21["loongarch64: move $r21"]
end
subgraph subGraph1["Target Architecture Detection"]
    X86_OFFSET["x86_64: mov offset VAR"]
    ARM_OFFSET["aarch64: movz #:abs_g0_nc:VAR"]
    RISCV_OFFSET["riscv: lui %hi(VAR), addi %lo(VAR)"]
    LOONG_OFFSET["loongarch64: lu12i.w %abs_hi20(VAR)"]
end
subgraph subGraph0["Code Generation Functions"]
    ARCH_GEN["Architecture Code Generation"]
    OFFSET["gen_offset()"]
    CURRENT_PTR["gen_current_ptr()"]
    READ_RAW["gen_read_current_raw()"]
    WRITE_RAW["gen_write_current_raw()"]
end

ARCH_GEN --> CURRENT_PTR
ARCH_GEN --> OFFSET
ARCH_GEN --> READ_RAW
ARCH_GEN --> WRITE_RAW
CURRENT_PTR --> ARM_TPIDR
CURRENT_PTR --> LOONG_R21
CURRENT_PTR --> RISCV_GP
CURRENT_PTR --> X86_GS
OFFSET --> ARM_OFFSET
OFFSET --> LOONG_OFFSET
OFFSET --> RISCV_OFFSET
OFFSET --> X86_OFFSET
```

**Sources:** [percpu_macros/src/arch.rs(L16 - L50)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L16-L50) [percpu_macros/src/arch.rs(L54 - L88)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L54-L88)

### Assembly Code Generation Patterns

The architecture-specific functions generate different assembly instruction sequences based on the target platform and operation type:

|Architecture|Offset Calculation|Current Pointer Access|Direct Read/Write|
| --- | --- | --- | --- |
|x86_64|mov {reg}, offset {symbol}|mov {reg}, gs:[offset __PERCPU_SELF_PTR]|mov gs:[offset {symbol}], {val}|
|AArch64|movz {reg}, #:abs_g0_nc:{symbol}|mrs {reg}, TPIDR_EL1|Not implemented|
|RISC-V|lui {reg}, %hi({symbol})|mv {reg}, gp|lui + add + load/store|
|LoongArch|lu12i.w {reg}, %abs_hi20({symbol})|move {reg}, $r21|lu12i.w + ori + load/store|

**Sources:** [percpu_macros/src/arch.rs(L94 - L181)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L94-L181) [percpu_macros/src/arch.rs(L187 - L263)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L187-L263)

## Feature-Based Code Variations

The pipeline adapts code generation based on feature flags, creating different implementations for various use cases and platform configurations.

### Feature Flag Processing

```mermaid
flowchart TD
subgraph subGraph2["Generated Code Variations"]
    GLOBAL_VAR["Global variable access"]
    SAFE_METHODS["Preemption-safe methods"]
    HYP_MODE["Hypervisor mode support"]
end
subgraph subGraph1["Code Generation Impact"]
    NAIVE_IMPL["naive.rs implementation"]
    GUARD_GEN["NoPreemptGuard generation"]
    TPIDR_EL2["TPIDR_EL2 register selection"]
end
subgraph subGraph0["Core Features"]
    FEATURES["Feature Configuration"]
    SP_NAIVE["sp-naive"]
    PREEMPT["preempt"]
    ARM_EL2["arm-el2"]
end

ARM_EL2 --> TPIDR_EL2
FEATURES --> ARM_EL2
FEATURES --> PREEMPT
FEATURES --> SP_NAIVE
GUARD_GEN --> SAFE_METHODS
NAIVE_IMPL --> GLOBAL_VAR
PREEMPT --> GUARD_GEN
SP_NAIVE --> NAIVE_IMPL
TPIDR_EL2 --> HYP_MODE
```

The feature-based variations are configured at [percpu_macros/Cargo.toml(L15 - L25)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/Cargo.toml#L15-L25) and affect code generation in several ways:

|Feature|Impact|Code Changes|
| --- | --- | --- |
|sp-naive|Single-processor fallback|Usesnaive.rsinstead ofarch.rs|
|preempt|Preemption safety|GeneratesNoPreemptGuardin safe methods|
|arm-el2|Hypervisor mode|UsesTPIDR_EL2instead ofTPIDR_EL1|

**Sources:** [percpu_macros/src/lib.rs(L59 - L60)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L59-L60) [percpu_macros/src/lib.rs(L94 - L98)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L94-L98) [percpu_macros/src/arch.rs(L55 - L61)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/arch.rs#L55-L61)

## Final Code Generation and Output

The pipeline concludes by assembling all generated components into the final token stream using the `quote!` macro.

### Output Structure Assembly

```mermaid
flowchart TD
subgraph subGraph1["Quote Assembly"]
    ASSEMBLY["Token Stream Assembly"]
    QUOTE_MACRO["quote! { ... }"]
    TOKEN_STREAM["proc_macro2::TokenStream"]
    PROC_MACRO["TokenStream::into()"]
end
subgraph subGraph0["Component Assembly"]
    COMPONENTS["Generated Components"]
    STORAGE_TOKENS["Storage variable tokens"]
    WRAPPER_TOKENS["Wrapper struct tokens"]
    IMPL_TOKENS["Implementation tokens"]
    INTERFACE_TOKENS["Interface tokens"]
end
OUTPUT["Final Output"]

ASSEMBLY --> OUTPUT
ASSEMBLY --> QUOTE_MACRO
COMPONENTS --> ASSEMBLY
COMPONENTS --> IMPL_TOKENS
COMPONENTS --> INTERFACE_TOKENS
COMPONENTS --> STORAGE_TOKENS
COMPONENTS --> WRAPPER_TOKENS
QUOTE_MACRO --> TOKEN_STREAM
TOKEN_STREAM --> PROC_MACRO
```

The final assembly process at [percpu_macros/src/lib.rs(L149 - L251)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L149-L251) combines:

1. **Storage Declaration**: `static mut __PERCPU_{name}: {type} = {init};` with `.percpu` section attribute
2. **Wrapper Struct**: Zero-sized struct with generated methods for access
3. **Implementation Block**: All the generated methods for the wrapper
4. **Public Interface**: `static {name}: {name}_WRAPPER = {name}_WRAPPER {};`

The complete transformation ensures that a simple `#[def_percpu] static VAR: T = init;` declaration becomes a comprehensive per-CPU data access system with architecture-optimized assembly code, type-safe interfaces, and optional preemption safety.

**Sources:** [percpu_macros/src/lib.rs(L149 - L252)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu_macros/src/lib.rs#L149-L252)