# Development Guide

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml)
> * [Cargo.lock](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.lock)

This guide provides essential information for developers working with or contributing to the page_table_multiarch library. It covers the development environment setup, multi-architecture compilation strategy, testing approach, and documentation generation processes.

For detailed build instructions and testing procedures, see [Building and Testing](/arceos-org/page_table_multiarch/5.1-building-and-testing). For contribution guidelines and coding standards, see [Contributing](/arceos-org/page_table_multiarch/5.2-contributing).

## Development Environment Overview

The page_table_multiarch project uses a sophisticated multi-architecture development approach that requires specific toolchain configurations and conditional compilation strategies. The project is designed to work across five different target architectures while maintaining a unified development experience.

### Development Architecture Matrix

```mermaid
flowchart TD
subgraph subGraph2["CI Pipeline Components"]
    FORMAT["cargo fmt"]
    CLIPPY["cargo clippy"]
    BUILD["cargo build"]
    TEST["cargo test"]
    DOC["cargo doc"]
end
subgraph subGraph1["Target Architectures"]
    LINUX["x86_64-unknown-linux-gnu"]
    NONE["x86_64-unknown-none"]
    RISCV["riscv64gc-unknown-none-elf"]
    ARM["aarch64-unknown-none-softfloat"]
    LOONG["loongarch64-unknown-none-softfloat"]
end
subgraph subGraph0["Development Environment"]
    RUST["Rust Nightly Toolchain"]
    COMPONENTS["rust-src, clippy, rustfmt"]
end

ARM --> BUILD
ARM --> CLIPPY
COMPONENTS --> ARM
COMPONENTS --> LINUX
COMPONENTS --> LOONG
COMPONENTS --> NONE
COMPONENTS --> RISCV
LINUX --> BUILD
LINUX --> CLIPPY
LINUX --> FORMAT
LINUX --> TEST
LOONG --> BUILD
LOONG --> CLIPPY
NONE --> BUILD
NONE --> CLIPPY
RISCV --> BUILD
RISCV --> CLIPPY
RUST --> COMPONENTS
TEST --> DOC
```

Sources: [.github/workflows/ci.yml(L10 - L32)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L10-L32)

### Workspace Dependency Structure

The development environment manages dependencies through a two-crate workspace with architecture-specific conditional compilation. The dependency resolution varies based on the target platform to include only necessary architecture support crates.

```mermaid
flowchart TD
subgraph subGraph2["Hardware Support Crates"]
    TOCK["tock-registers v0.9.0"]
    RAW_CPUID["raw-cpuid v10.7.0"]
    BIT_FIELD["bit_field v0.10.2"]
    BIT_FIELD_2["bit_field v0.10.2"]
    VOLATILE["volatile v0.4.6"]
    CRITICAL["critical-section v1.2.0"]
    EMBEDDED["embedded-hal v1.0.0"]
end
subgraph subGraph1["page_table_entry Dependencies"]
    PTE_MEM["memory_addr v0.3.1"]
    AARCH64["aarch64-cpu v10.0.0"]
    BITFLAGS["bitflags v2.8.0"]
    X86_64["x86_64 v0.15.2"]
end
subgraph subGraph0["page_table_multiarch Dependencies"]
    PTM["page_table_multiarch v0.5.3"]
    LOG["log v0.4.25"]
    MEMORY["memory_addr v0.3.1"]
    PTE["page_table_entry v0.5.3"]
    RISCV_CRATE["riscv v0.12.1"]
    X86_CRATE["x86 v0.52.0"]
end

AARCH64 --> TOCK
PTE --> AARCH64
PTE --> BITFLAGS
PTE --> PTE_MEM
PTE --> X86_64
PTM --> LOG
PTM --> MEMORY
PTM --> PTE
PTM --> RISCV_CRATE
PTM --> X86_CRATE
RISCV_CRATE --> CRITICAL
RISCV_CRATE --> EMBEDDED
X86_CRATE --> BIT_FIELD
X86_CRATE --> RAW_CPUID
```

Sources: [Cargo.lock(L57 - L75)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.lock#L57-L75) [Cargo.lock(L5 - L149)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/Cargo.lock#L5-L149)

## Multi-Architecture Development Strategy

The project employs a conditional compilation strategy that allows development and testing across multiple architectures while maintaining code clarity and avoiding unnecessary dependencies for specific targets.

### Conditional Compilation System

The build system uses Cargo's target-specific dependencies and feature flags to include only the necessary architecture support:

|Target Architecture|Included Dependencies|Special Features|
| --- | --- | --- |
|x86_64-unknown-linux-gnu|x86,x86_64, full test suite|Unit testing enabled|
|x86_64-unknown-none|x86,x86_64|Bare metal support|
|riscv64gc-unknown-none-elf|riscv|RISC-V Sv39/Sv48 support|
|aarch64-unknown-none-softfloat|aarch64-cpu|ARM EL2 support available|
|loongarch64-unknown-none-softfloat|Basic LoongArch support|Custom PWC configuration|

### Documentation Build Configuration

Documentation generation requires special configuration to include all architecture-specific code regardless of the build target:

```mermaid
flowchart TD
subgraph subGraph1["Architecture Inclusion"]
    ALL_ARCH["All architecture modules included"]
    X86_DOCS["x86_64 documentation"]
    ARM_DOCS["AArch64 documentation"]
    RISCV_DOCS["RISC-V documentation"]
    LOONG_DOCS["LoongArch documentation"]
end
subgraph subGraph0["Documentation Build Process"]
    DOC_ENV["RUSTFLAGS: --cfg doc"]
    DOC_FLAGS["RUSTDOCFLAGS: -Zunstable-options --enable-index-page"]
    DOC_BUILD["cargo doc --no-deps --all-features"]
end

ALL_ARCH --> ARM_DOCS
ALL_ARCH --> LOONG_DOCS
ALL_ARCH --> RISCV_DOCS
ALL_ARCH --> X86_DOCS
DOC_BUILD --> ALL_ARCH
DOC_ENV --> DOC_BUILD
DOC_FLAGS --> DOC_BUILD
```

Sources: [.github/workflows/ci.yml(L40 - L49)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L40-L49)

## CI/CD Pipeline Architecture

The continuous integration system validates code across all supported architectures using a matrix build strategy. This ensures that changes work correctly across the entire supported platform ecosystem.

### CI Matrix Strategy

```mermaid
flowchart TD
subgraph subGraph3["Documentation Pipeline"]
    DOC_BUILD["cargo doc --no-deps --all-features"]
    PAGES_DEPLOY["GitHub Pages Deployment"]
end
subgraph subGraph2["Validation Steps"]
    VERSION_CHECK["rustc --version --verbose"]
    FORMAT_CHECK["cargo fmt --all -- --check"]
    CLIPPY_CHECK["cargo clippy --target TARGET --all-features"]
    BUILD_STEP["cargo build --target TARGET --all-features"]
    UNIT_TEST["cargo test --target x86_64-unknown-linux-gnu"]
end
subgraph subGraph1["Build Matrix"]
    NIGHTLY["Rust Nightly Toolchain"]
    TARGETS["5 Target Architectures"]
end
subgraph subGraph0["CI Trigger Events"]
    PUSH["git push"]
    PR["pull_request"]
end

BUILD_STEP --> UNIT_TEST
CLIPPY_CHECK --> BUILD_STEP
DOC_BUILD --> PAGES_DEPLOY
FORMAT_CHECK --> CLIPPY_CHECK
NIGHTLY --> TARGETS
PR --> NIGHTLY
PUSH --> NIGHTLY
TARGETS --> VERSION_CHECK
UNIT_TEST --> DOC_BUILD
VERSION_CHECK --> FORMAT_CHECK
```

Sources: [.github/workflows/ci.yml(L1 - L33)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L1-L33) [.github/workflows/ci.yml(L34 - L56)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L34-L56)

### Quality Gates

The CI pipeline enforces several quality gates that must pass before code integration:

|Check Type|Command|Scope|Failure Behavior|
| --- | --- | --- | --- |
|Format|cargo fmt --all -- --check|All files|Hard failure|
|Clippy|cargo clippy --target TARGET --all-features|Per target|Hard failure|
|Build|cargo build --target TARGET --all-features|Per target|Hard failure|
|Unit Tests|cargo test --target x86_64-unknown-linux-gnu|Linux only|Hard failure|
|Documentation|cargo doc --no-deps --all-features|All features|Soft failure on non-default branch|

## Development Workflow Integration

The development environment integrates multiple tools and processes to maintain code quality across architecture boundaries:

### Toolchain Requirements

* **Rust Toolchain**: Nightly required for unstable features and cross-compilation support
* **Components**: `rust-src` for cross-compilation, `clippy` for linting, `rustfmt` for formatting
* **Targets**: All five supported architectures must be installed for comprehensive testing

### Feature Flag Usage

The project uses Cargo feature flags to control compilation of architecture-specific code:

```mermaid
flowchart TD
subgraph subGraph1["Compilation Outcomes"]
    FULL_BUILD["Complete architecture support"]
    DOC_BUILD["Documentation with all architectures"]
    ARM_PRIVILEGE["ARM EL2 privilege level support"]
end
subgraph subGraph0["Feature Control"]
    ALL_FEATURES["--all-features"]
    DOC_CFG["--cfg doc"]
    ARM_EL2["arm-el2 feature"]
end

ALL_FEATURES --> FULL_BUILD
ARM_EL2 --> ARM_PRIVILEGE
DOC_CFG --> DOC_BUILD
FULL_BUILD --> DOC_BUILD
```

Sources: [.github/workflows/ci.yml(L25 - L27)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L25-L27) [.github/workflows/ci.yml(L42)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L42-L42)

The development environment is designed to handle the complexity of multi-architecture support while providing developers with clear feedback about compatibility and correctness across all supported platforms. The CI pipeline ensures that every change is validated against the complete matrix of supported architectures before integration.