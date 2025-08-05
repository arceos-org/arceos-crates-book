# Development and Project Setup

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml)
> * [Cargo.toml](https://github.com/arceos-org/memory_set/blob/73b51e2b/Cargo.toml)

This document provides essential information for developers working on or with the `memory_set` crate, covering project dependencies, build configuration, supported targets, and development workflows. For implementation details and API usage, see [Implementation Details](/arceos-org/memory_set/2-implementation-details) and [Usage and Examples](/arceos-org/memory_set/3-usage-and-examples).

## Project Overview and Structure

The `memory_set` crate is designed as a foundational component within the ArceOS ecosystem, providing `no_std` compatible memory mapping management capabilities. The project follows standard Rust conventions with additional considerations for embedded and OS development environments.

**Project Structure and Dependencies**

```mermaid
flowchart TD
subgraph Licensing["Licensing"]
    LIC["GPL-3.0-or-later OR Apache-2.0 OR MulanPSL-2.0"]
end
subgraph Categorization["Categorization"]
    KW["keywords: [arceos, virtual-memory, memory-area, mmap]"]
    CAT["categories: [os, memory-management, no-std]"]
end
subgraph subGraph2["External Dependencies"]
    MA["memory_addr = 0.2"]
end
subgraph subGraph1["Package Metadata"]
    PN["name: memory_set"]
    PV["version: 0.1.0"]
    PE["edition: 2021"]
    PA["author: Yuekai Jia"]
end
subgraph subGraph0["Project Configuration"]
    CT["Cargo.toml"]
    CI[".github/workflows/ci.yml"]
end

CT --> CAT
CT --> KW
CT --> LIC
CT --> MA
CT --> PA
CT --> PE
CT --> PN
CT --> PV
```

Sources: [Cargo.toml(L1 - L16)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/Cargo.toml#L1-L16)

The crate maintains minimal external dependencies, requiring only the `memory_addr` crate for address type definitions. This design choice supports the `no_std` environment requirement and reduces compilation complexity.

|Configuration|Value|Purpose|
| --- | --- | --- |
|Edition|2021|Modern Rust language features|
|Categories|os,memory-management,no-std|Ecosystem positioning|
|License|Triple-licensed|Compatibility with various projects|
|Repository|github.com/arceos-org/memory_set|Source code location|
|Documentation|docs.rs/memory_set|API documentation|

Sources: [Cargo.toml(L1 - L16)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/Cargo.toml#L1-L16)

## Build Configuration and Target Support

The project supports multiple target architectures, reflecting its intended use in diverse embedded and OS development scenarios. The build configuration accommodates both hosted and bare-metal environments.

**Supported Target Architectures**

```mermaid
flowchart TD
subgraph subGraph2["Bare Metal Targets"]
    X86N["x86_64-unknown-none"]
    RISCV["riscv64gc-unknown-none-elf"]
    ARM64["aarch64-unknown-none-softfloat"]
end
subgraph subGraph1["Hosted Targets"]
    X86L["x86_64-unknown-linux-gnu"]
end
subgraph subGraph0["CI Target Matrix"]
    RT["nightly toolchain"]
end

RT --> ARM64
RT --> RISCV
RT --> X86L
RT --> X86N
```

Sources: [.github/workflows/ci.yml(L11 - L12)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L11-L12)

The build process includes several validation steps:

|Step|Command|Purpose|
| --- | --- | --- |
|Format Check|cargo fmt --all -- --check|Code style enforcement|
|Linting|cargo clippy --target TARGET --all-features|Static analysis|
|Build|cargo build --target TARGET --all-features|Compilation verification|
|Testing|cargo test --target TARGET|Unit test execution|

Sources: [.github/workflows/ci.yml(L23 - L30)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L23-L30)

### Toolchain Requirements

The project requires the nightly Rust toolchain with specific components:

* `rust-src`: Required for no_std target compilation
* `clippy`: Static analysis and linting
* `rustfmt`: Code formatting enforcement

**Toolchain Setup Process**

```mermaid
sequenceDiagram
    participant Developer as Developer
    participant CISystem as CI System
    participant RustToolchain as Rust Toolchain
    participant TargetBuild as Target Build

    Developer ->> CISystem: "Push/PR trigger"
    CISystem ->> RustToolchain: "Install nightly toolchain"
    RustToolchain ->> CISystem: "rust-src, clippy, rustfmt components"
    CISystem ->> RustToolchain: "rustc --version --verbose"
    loop "For each target"
        CISystem ->> TargetBuild: "cargo fmt --check"
        CISystem ->> TargetBuild: "cargo clippy --target TARGET"
        CISystem ->> TargetBuild: "cargo build --target TARGET"
    alt "x86_64-unknown-linux-gnu only"
        CISystem ->> TargetBuild: "cargo test --target TARGET"
    end
    end
```

Sources: [.github/workflows/ci.yml(L14 - L30)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L14-L30)

## Development Workflow and CI Pipeline

The continuous integration system enforces code quality standards and validates functionality across all supported targets. The workflow is designed to catch issues early and maintain consistency.

### CI Job Configuration

The CI pipeline consists of two primary jobs:

1. **Main CI Job** (`ci`): Validates code across all target architectures
2. **Documentation Job** (`doc`): Builds and deploys API documentation

**CI Pipeline Architecture**

```mermaid
flowchart TD
subgraph subGraph5["Documentation Job"]
    DOCBUILD["cargo doc --no-deps"]
    DEPLOY["Deploy to gh-pages"]
end
subgraph subGraph4["Validation Steps"]
    FMT["cargo fmt --check"]
    CLIP["cargo clippy"]
    BUILD["cargo build"]
    TEST["cargo test (linux only)"]
end
subgraph subGraph3["CI Job Matrix"]
    subgraph Targets["Targets"]
        T1["x86_64-unknown-linux-gnu"]
        T2["x86_64-unknown-none"]
        T3["riscv64gc-unknown-none-elf"]
        T4["aarch64-unknown-none-softfloat"]
    end
    subgraph Toolchain["Toolchain"]
        NIGHTLY["nightly"]
    end
end
subgraph subGraph0["Trigger Events"]
    PUSH["push events"]
    PR["pull_request events"]
end
CI["CI"]

BUILD --> TEST
CI --> DOCBUILD
CLIP --> BUILD
DOCBUILD --> DEPLOY
FMT --> CLIP
NIGHTLY --> T1
NIGHTLY --> T2
NIGHTLY --> T3
NIGHTLY --> T4
PR --> CI
PUSH --> CI
T1 --> FMT
T2 --> FMT
T3 --> FMT
T4 --> FMT
```

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L1-L56)

### Documentation Generation

The documentation workflow includes specialized configuration for maintaining high-quality API documentation:

* **RUSTDOCFLAGS**: `-D rustdoc::broken_intra_doc_links -D missing-docs`
* **Deployment**: Automatic GitHub Pages deployment on main branch
* **Index Generation**: Automatic redirect to crate documentation

Sources: [.github/workflows/ci.yml(L40 - L48)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L40-L48)

### Testing Strategy

Testing is performed only on the `x86_64-unknown-linux-gnu` target due to the hosted environment requirements for test execution. The test command includes `--nocapture` to display output from test functions.

**Test Execution Conditions**

```mermaid
flowchart TD
START["CI Job Start"]
CHECK_TARGET["Target == x86_64-unknown-linux-gnu?"]
RUN_TESTS["cargo test --target TARGET -- --nocapture"]
SKIP_TESTS["Skip unit tests"]
END["Job Complete"]

CHECK_TARGET --> RUN_TESTS
CHECK_TARGET --> SKIP_TESTS
RUN_TESTS --> END
SKIP_TESTS --> END
START --> CHECK_TARGET
```

Sources: [.github/workflows/ci.yml(L28 - L30)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L28-L30)

## ArceOS Ecosystem Integration

The `memory_set` crate is positioned as a foundational component within the broader ArceOS operating system project. This integration influences several design decisions:

* **Homepage**: Points to the main ArceOS project (`github.com/arceos-org/arceos`)
* **Keywords**: Include `arceos` for discoverability within the ecosystem
* **License**: Triple-licensing supports integration with various ArceOS components
* **No-std Compatibility**: Essential for kernel-level memory management

Sources: [Cargo.toml(L8 - L12)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/Cargo.toml#L8-L12)

The minimal dependency footprint and broad target support make this crate suitable for integration across different ArceOS configurations and deployment scenarios.