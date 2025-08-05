# Overview

> **Relevant source files**
> * [Cargo.toml](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml)
> * [README.md](https://github.com/arceos-org/crate_interface/blob/73011a44/README.md)

## Purpose and Scope

The `crate_interface` crate is a procedural macro library that enables cross-crate trait interfaces in Rust. It provides a solution for defining trait interfaces in one crate while allowing implementations and usage in separate crates, effectively solving circular dependency problems between crates. This document covers the high-level architecture, core concepts, and integration patterns within the ArceOS ecosystem.

For detailed usage instructions, see [Getting Started](/arceos-org/crate_interface/2-getting-started). For comprehensive macro documentation, see [Macro Reference](/arceos-org/crate_interface/3-macro-reference). For implementation details, see [Architecture and Internals](/arceos-org/crate_interface/4-architecture-and-internals).

*Sources: [Cargo.toml(L1 - L22)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml#L1-L22) [README.md(L1 - L85)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/README.md#L1-L85)*

## Core Problem and Solution

The crate addresses the fundamental challenge of **circular dependencies** in Rust crate ecosystems. Traditional trait definitions create tight coupling between interface definitions and their implementations, preventing modular plugin architectures.

The `crate_interface` solution employs a three-phase approach using procedural macros that generate `extern "Rust"` function declarations and implementations with specific symbol naming conventions. This allows the Rust linker to resolve cross-crate trait method calls without requiring direct crate dependencies.

```mermaid
flowchart TD
subgraph subGraph1["crate_interface Solution"]
    G["Generated #[export_name] functions"]
    H["Crate C: call_interface!"]
    I["Safe extern function calls"]
    subgraph subGraph0["Traditional Approach Problem"]
        D["Crate A: #[def_interface]"]
        E["Generated extern declarations"]
        F["Crate B: #[impl_interface]"]
        A["Crate A: Interface"]
        B["Crate B: Implementation"]
        C["Crate C: Consumer"]
    end
end

A --> B
A --> C
B --> C
C --> A
D --> E
E --> G
F --> G
H --> I
I --> G
```

*Sources: [README.md(L7 - L10)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/README.md#L7-L10) [Cargo.toml(L6)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml#L6-L6)*

## Three-Macro System Architecture

The crate implements a coordinated system of three procedural macros that work together to enable cross-crate trait interfaces:

```mermaid
flowchart TD
subgraph subGraph3["Runtime Linking"]
    M["Rust Linker"]
end
subgraph subGraph2["call_interface Phase"]
    I["call_interface!"]
    J["HelloIf::hello macro call"]
    K["unsafe extern function call"]
    L["__HelloIf_mod::__HelloIf_hello"]
end
subgraph subGraph1["impl_interface Phase"]
    E["#[impl_interface]"]
    F["impl HelloIf for HelloIfImpl"]
    G["#[export_name] extern fn"]
    H["__HelloIf_hello symbol"]
end
subgraph subGraph0["def_interface Phase"]
    A["#[def_interface]"]
    B["trait HelloIf"]
    C["__HelloIf_mod module"]
    D["extern fn __HelloIf_hello"]
end

A --> B
B --> C
C --> D
D --> M
E --> F
F --> G
G --> H
H --> M
I --> J
J --> K
K --> L
L --> M
```

*Sources: [README.md(L13 - L40)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/README.md#L13-L40) [README.md(L44 - L85)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/README.md#L44-L85)*

## Generated Code Flow

The macro system transforms high-level trait definitions into low-level extern function interfaces that can be linked across crate boundaries:

```

```

*Sources: [README.md(L46 - L85)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/README.md#L46-L85)*

## Integration with ArceOS Ecosystem

The `crate_interface` crate is designed as a foundational component within the ArceOS modular operating system architecture. It enables plugin-style extensibility and cross-crate interfaces essential for kernel modularity:

|Feature|Purpose|ArceOS Use Case|
| --- | --- | --- |
|No-std compatibility|Embedded/kernel environments|Bare-metal kernel modules|
|Cross-crate interfaces|Plugin architecture|Device drivers, file systems|
|Symbol-based linking|Runtime module loading|Dynamic kernel extensions|
|Zero-overhead abstractions|Performance-critical code|Kernel syscall interfaces|

```mermaid
flowchart TD
subgraph subGraph1["Build Targets"]
    J["x86_64-unknown-none"]
    K["riscv64gc-unknown-none-elf"]
    L["aarch64-unknown-none-softfloat"]
end
subgraph subGraph0["ArceOS Architecture"]
    A["Kernel Core"]
    B["crate_interface"]
    C["Driver Interface Traits"]
    D["FS Interface Traits"]
    E["Network Interface Traits"]
    F["Device Drivers"]
    G["File Systems"]
    H["Network Stack"]
    I["Applications"]
end

A --> B
B --> C
B --> D
B --> E
B --> J
B --> K
B --> L
F --> C
G --> D
H --> E
I --> C
I --> D
I --> E
```

*Sources: [Cargo.toml(L8)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml#L8-L8) [Cargo.toml(L11 - L12)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml#L11-L12)*

## Key Design Principles

The crate follows several core design principles that make it suitable for system-level programming:

* **Symbol-based Decoupling**: Uses extern function symbols instead of direct trait object vtables
* **Compile-time Safety**: Procedural macros ensure type safety while generating unsafe extern calls
* **Zero Runtime Overhead**: Direct function calls with no dynamic dispatch or allocations
* **Cross-platform Compatibility**: Works across multiple target architectures and toolchains
* **No-std First**: Designed for embedded and kernel environments without standard library dependencies

*Sources: [Cargo.toml(L12)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml#L12-L12) [Cargo.toml(L15 - L18)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml#L15-L18)*