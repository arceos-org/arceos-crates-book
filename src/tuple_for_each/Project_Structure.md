# Project Structure

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.gitignore)
> * [Cargo.toml](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/Cargo.toml)

This document describes the architectural organization of the `tuple_for_each` crate, including its dependencies, build configuration, and development infrastructure. The project is a procedural macro library designed to generate iteration utilities for tuple structs, with particular emphasis on cross-platform compatibility for embedded systems development.

For details about the core macro implementation, see [3.1](/arceos-org/tuple_for_each/3.1-derive-macro-processing). For information about the CI/CD pipeline specifics, see [4.2](/arceos-org/tuple_for_each/4.2-cicd-pipeline).

## Crate Architecture

The `tuple_for_each` crate follows a standard Rust procedural macro library structure, with the core implementation residing in `src/lib.rs` and supporting infrastructure for multi-platform builds and documentation.

### Crate Configuration

```mermaid
flowchart TD
subgraph subGraph3["ArceOS Ecosystem"]
    HOMEPAGE["Homepage: arceos-org/arceos"]
    REPO["Repository: tuple_for_each"]
    DOCS["Documentation: docs.rs"]
end
subgraph Dependencies["Dependencies"]
    SYN["syn = '2.0'"]
    QUOTE["quote = '1.0'"]
    PM2["proc-macro2 = '1.0'"]
end
subgraph subGraph1["Crate Type"]
    PROC["proc-macro = true"]
    LIB["[lib] Configuration"]
end
subgraph subGraph0["Package Metadata"]
    PM["tuple_for_eachv0.1.0Edition 2021"]
    AUTH["Author: Yuekai Jia"]
    LIC["Triple License:GPL-3.0 | Apache-2.0 | MulanPSL-2.0"]
end

HOMEPAGE --> REPO
LIB --> PM2
LIB --> QUOTE
LIB --> SYN
PM --> HOMEPAGE
PM --> PROC
PROC --> LIB
REPO --> DOCS
```

**Crate Configuration Details**

The crate is configured as a procedural macro library through the `proc-macro = true` setting in [Cargo.toml(L19 - L20)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/Cargo.toml#L19-L20) This enables the crate to export procedural macros that operate at compile time. The package metadata indicates this is part of the ArceOS project ecosystem, focusing on systems programming and embedded development.

**Sources:** [Cargo.toml(L1 - L21)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/Cargo.toml#L1-L21)

### Dependency Architecture

```mermaid
flowchart TD
subgraph subGraph2["Generated Artifacts"]
    MACROS["_for_each!_enumerate!"]
    METHODS["len()is_empty()"]
end
subgraph subGraph1["Procedural Macro Stack"]
    SYN_DEP["syn 2.0AST Parsing"]
    QUOTE_DEP["quote 1.0Code Generation"]
    PM2_DEP["proc-macro2 1.0Token Management"]
end
subgraph subGraph0["tuple_for_each Crate"]
    DERIVE["TupleForEachDerive Macro"]
    IMPL["impl_for_each()Core Logic"]
end

DERIVE --> PM2_DEP
DERIVE --> QUOTE_DEP
DERIVE --> SYN_DEP
IMPL --> QUOTE_DEP
IMPL --> SYN_DEP
PM2_DEP --> QUOTE_DEP
QUOTE_DEP --> MACROS
QUOTE_DEP --> METHODS
SYN_DEP --> IMPL
```

**Dependency Roles**

|Dependency|Version|Purpose|
| --- | --- | --- |
|syn|2.0|Parsing Rust syntax trees from macro input tokens|
|quote|1.0|Generating Rust code from templates and interpolation|
|proc-macro2|1.0|Low-level token stream manipulation and span handling|

The dependency selection follows Rust procedural macro best practices, using the latest stable versions of the core macro development libraries.

**Sources:** [Cargo.toml(L14 - L17)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/Cargo.toml#L14-L17)

## Build Matrix and Target Support

### Multi-Platform Build Configuration

```mermaid
flowchart TD
subgraph subGraph2["Build Strategy"]
    FULL_TEST["Full CI PipelineFormat + Lint + Build + Test"]
    BUILD_ONLY["Build VerificationFormat + Lint + Build"]
end
subgraph subGraph1["Target Platforms"]
    X86_LINUX["x86_64-unknown-linux-gnuDevelopment & Testing"]
    X86_NONE["x86_64-unknown-noneBare Metal x86"]
    RISCV["riscv64gc-unknown-none-elfRISC-V Embedded"]
    ARM["aarch64-unknown-none-softfloatARM64 Embedded"]
end
subgraph subGraph0["Rust Toolchain"]
    NIGHTLY["nightlyRequired Toolchain"]
    COMPONENTS["Components:rust-src, clippy, rustfmt"]
end

ARM --> BUILD_ONLY
COMPONENTS --> ARM
COMPONENTS --> RISCV
COMPONENTS --> X86_LINUX
COMPONENTS --> X86_NONE
NIGHTLY --> COMPONENTS
RISCV --> BUILD_ONLY
X86_LINUX --> FULL_TEST
X86_NONE --> BUILD_ONLY
```

**Target Platform Strategy**

The build matrix demonstrates the crate's focus on embedded and systems programming:

* **Primary Development**: `x86_64-unknown-linux-gnu` with full testing support
* **Bare Metal x86**: `x86_64-unknown-none` for bootloader and kernel development
* **RISC-V Embedded**: `riscv64gc-unknown-none-elf` for RISC-V microcontrollers
* **ARM64 Embedded**: `aarch64-unknown-none-softfloat` for ARM embedded systems

Testing is restricted to the Linux target due to the embedded nature of other platforms, which typically lack standard library support required for test execution.

**Sources:** [.github/workflows/ci.yml(L10 - L12)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L10-L12) [.github/workflows/ci.yml(L28 - L30)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L28-L30)

## Development Infrastructure

### CI/CD Pipeline Architecture

```mermaid
flowchart TD
subgraph subGraph3["Documentation Pipeline"]
    DOC_BUILD["cargo doc --no-deps"]
    REDIRECT["Generate index.html redirect"]
    DEPLOY["GitHub Pages deployment"]
end
subgraph subGraph2["CI Pipeline Steps"]
    CHECKOUT["actions/checkout@v4"]
    TOOLCHAIN["dtolnay/rust-toolchain@nightly"]
    FORMAT["cargo fmt --check"]
    CLIPPY["cargo clippy"]
    BUILD["cargo build"]
    TEST["cargo test(x86_64-linux only)"]
end
subgraph subGraph1["CI Job Matrix"]
    CI_JOB["ci jobMulti-target builds"]
    DOC_JOB["doc jobDocumentation"]
end
subgraph subGraph0["Trigger Events"]
    PUSH["git push"]
    PR["pull_request"]
end

BUILD --> TEST
CHECKOUT --> TOOLCHAIN
CI_JOB --> CHECKOUT
CLIPPY --> BUILD
DOC_BUILD --> REDIRECT
DOC_JOB --> DOC_BUILD
FORMAT --> CLIPPY
PR --> CI_JOB
PUSH --> CI_JOB
PUSH --> DOC_JOB
REDIRECT --> DEPLOY
TOOLCHAIN --> FORMAT
```

**Quality Assurance Steps**

The CI pipeline enforces code quality through multiple verification stages:

1. **Format Checking**: `cargo fmt --all -- --check` ensures consistent code style
2. **Linting**: `cargo clippy` with custom configuration excluding `new_without_default` warnings
3. **Multi-target Building**: Verification across all supported platforms
4. **Testing**: Unit tests executed only on `x86_64-unknown-linux-gnu`

**Sources:** [.github/workflows/ci.yml(L22 - L30)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L22-L30)

### Documentation Deployment

The documentation system automatically builds and deploys to GitHub Pages on pushes to the default branch. The deployment includes:

* **API Documentation**: Generated via `cargo doc --no-deps --all-features`
* **Redirect Setup**: Automatic index.html generation for seamless navigation
* **Single Commit Deployment**: Clean deployment strategy to the `gh-pages` branch

**Sources:** [.github/workflows/ci.yml(L44 - L55)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L44-L55)

## Development Environment

### Required Toolchain Components

```mermaid
flowchart TD
subgraph subGraph2["Target Installation"]
    TARGETS["rustup target addAll matrix targets"]
end
subgraph subGraph1["Required Components"]
    RUST_SRC["rust-srcFor no_std targets"]
    CLIPPY["clippyLinting"]
    RUSTFMT["rustfmtCode formatting"]
end
subgraph subGraph0["Rust Nightly Toolchain"]
    RUSTC["rustc nightly"]
    CARGO["cargo"]
end

CARGO --> CLIPPY
CARGO --> RUSTFMT
RUSTC --> RUST_SRC
RUST_SRC --> TARGETS
```

**Development Setup Requirements**

* **Rust Nightly**: Required for advanced procedural macro features
* **rust-src Component**: Essential for building `no_std` embedded targets
* **Cross-compilation Support**: All target platforms must be installed via `rustup`

**Sources:** [.github/workflows/ci.yml(L15 - L19)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L15-L19)

### Project File Structure

|File/Directory|Purpose|
| --- | --- |
|Cargo.toml|Package configuration and dependencies|
|src/lib.rs|Core procedural macro implementation|
|tests/|Integration tests for macro functionality|
|.github/workflows/|CI/CD pipeline definitions|
|.gitignore|Version control exclusions|

The project follows standard Rust library conventions with emphasis on procedural macro development patterns.

**Sources:** [.gitignore(L1 - L5)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.gitignore#L1-L5)