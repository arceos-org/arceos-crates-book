# Development Guide

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.gitignore)
> * [Cargo.toml](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/Cargo.toml)

## Purpose and Scope

This guide provides essential information for developers contributing to the `linked_list_r4l` crate. It covers build setup, testing procedures, CI/CD pipeline configuration, and project structure. For information about the library's APIs and usage patterns, see [API Reference](/arceos-org/linked_list_r4l/4-api-reference). For architectural concepts and design principles, see [Architecture Overview](/arceos-org/linked_list_r4l/3-architecture-overview) and [Core Concepts](/arceos-org/linked_list_r4l/5-core-concepts).

## Prerequisites and Environment Setup

The `linked_list_r4l` crate requires specific toolchain components and supports multiple target architectures for embedded and systems programming use cases.

### Required Toolchain

|Component|Version|Purpose|
| --- | --- | --- |
|Rust Toolchain|nightly|Required for advanced features|
|rust-src|Latest|Source code for cross-compilation|
|clippy|Latest|Linting and code analysis|
|rustfmt|Latest|Code formatting|

### Supported Target Architectures

The project supports multiple target architectures as defined in the CI pipeline:

```mermaid
flowchart TD
subgraph subGraph1["Development Activities"]
    UnitTests["Unit Testingcargo test"]
    CrossBuild["Cross Compilationcargo build"]
    Clippy["Static Analysiscargo clippy"]
    Format["Code Formattingcargo fmt"]
end
subgraph subGraph0["Supported Targets"]
    HostTarget["x86_64-unknown-linux-gnu\Host Development\"]
    BareMetal["x86_64-unknown-none\Bare Metal x86_64\"]
    RiscV["riscv64gc-unknown-none-elf\RISC-V Embedded\"]
    ARM["aarch64-unknown-none-softfloat\ARM64 Embedded\"]
end

ARM --> CrossBuild
BareMetal --> CrossBuild
Clippy --> Format
CrossBuild --> Clippy
HostTarget --> CrossBuild
HostTarget --> UnitTests
RiscV --> CrossBuild
UnitTests --> Clippy
```

Sources: [.github/workflows/ci.yml(L12)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml#L12-L12) [.github/workflows/ci.yml(L15 - L19)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml#L15-L19)

## Project Structure

The `linked_list_r4l` crate follows a standard Rust library structure with specialized CI/CD configuration for embedded systems development.

### Repository Layout

```mermaid
flowchart TD
subgraph subGraph2["Build Outputs"]
    DocDir["target/doc/\Generated Documentation\"]
    BuildDir["target/[target]/\Compiled Artifacts\"]
end
subgraph subGraph1["GitHub Integration"]
    WorkflowsDir[".github/workflows/\CI/CD Configuration\"]
    CIYml["ci.yml\Build Pipeline\"]
    GHPages["gh-pages\Documentation Deployment\"]
end
subgraph subGraph0["Root Directory"]
    CargoToml["Cargo.toml\Package Configuration\"]
    GitIgnore[".gitignore\Version Control\"]
    SrcDir["src/\Source Code\"]
    TargetDir["target/\Build Artifacts\"]
end

CIYml --> DocDir
CIYml --> TargetDir
CargoToml --> SrcDir
DocDir --> GHPages
TargetDir --> BuildDir
WorkflowsDir --> CIYml
```

Sources: [Cargo.toml(L1 - L15)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/Cargo.toml#L1-L15) [.gitignore(L1 - L5)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.gitignore#L1-L5) [.github/workflows/ci.yml(L1)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml#L1-L1)

## Build System Configuration

The crate is configured as a `no-std` library with no external dependencies, making it suitable for embedded environments.

### Package Metadata

The project configuration emphasizes systems programming and embedded compatibility:

|Property|Value|Significance|
| --- | --- | --- |
|name|linked_list_r4l|Crate identifier|
|version|0.2.1|Current release version|
|edition|2021|Rust edition with latest features|
|categories|["no-std", "rust-patterns"]|Embedded and pattern library|
|license|GPL-2.0-or-later|Open source license|

### Dependency Management

```mermaid
flowchart TD
subgraph subGraph2["Internal Modules"]
    LibRS["lib.rs\Public API\"]
    LinkedListRS["linked_list.rs\High-Level Interface\"]
    RawListRS["raw_list.rs\Low-Level Operations\"]
end
subgraph subGraph1["Standard Library"]
    NoStd["#![no_std]\Core Library Only\"]
    CoreCrate["core::\Essential Types\"]
end
subgraph subGraph0["External Dependencies"]
    NoDeps["[dependencies]\No External Crates\"]
end

CoreCrate --> LibRS
LibRS --> LinkedListRS
LibRS --> RawListRS
NoDeps --> NoStd
NoStd --> CoreCrate
```

Sources: [Cargo.toml(L14 - L15)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/Cargo.toml#L14-L15) [Cargo.toml(L12)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/Cargo.toml#L12-L12)

## Testing Framework

The testing infrastructure supports both unit testing and cross-compilation verification across multiple target architectures.

### Test Execution Strategy

```mermaid
sequenceDiagram
    participant Developer as "Developer"
    participant CIPipeline as "CI Pipeline"
    participant BuildMatrix as "Build Matrix"
    participant x86_64unknownlinuxgnu as "x86_64-unknown-linux-gnu"
    participant EmbeddedTargets as "Embedded Targets"

    Developer ->> CIPipeline: "Push/PR Trigger"
    CIPipeline ->> BuildMatrix: "Initialize Build Matrix"
    Note over BuildMatrix,EmbeddedTargets: Parallel Execution
    BuildMatrix ->> x86_64unknownlinuxgnu: "cargo test --target x86_64-unknown-linux-gnu"
    BuildMatrix ->> EmbeddedTargets: "cargo build --target [embedded-target]"
    x86_64unknownlinuxgnu ->> x86_64unknownlinuxgnu: "Run Unit Tests with --nocapture"
    EmbeddedTargets ->> EmbeddedTargets: "Verify Compilation Only"
    x86_64unknownlinuxgnu -->> CIPipeline: "Test Results"
    EmbeddedTargets -->> CIPipeline: "Build Verification"
    CIPipeline -->> Developer: "Pipeline Status"
```

### Test Configuration

Unit tests are executed only on the host target (`x86_64-unknown-linux-gnu`) while other targets verify compilation compatibility:

```markdown
# Host target with unit tests
cargo test --target x86_64-unknown-linux-gnu -- --nocapture

# Cross-compilation verification
cargo build --target x86_64-unknown-none --all-features
cargo build --target riscv64gc-unknown-none-elf --all-features  
cargo build --target aarch64-unknown-none-softfloat --all-features
```

Sources: [.github/workflows/ci.yml(L28 - L30)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml#L28-L30) [.github/workflows/ci.yml(L26 - L27)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml#L26-L27)

## CI/CD Pipeline Architecture

The continuous integration system implements a comprehensive quality assurance workflow with parallel job execution and automated documentation deployment.

### Pipeline Workflow

```mermaid
flowchart TD
subgraph subGraph3["Documentation Pipeline"]
    DocBuild["cargo doc --no-deps --all-features"]
    DocDeploy["JamesIves/github-pages-deploy-action@v4"]
    GHPages["GitHub Pages Deployment"]
end
subgraph subGraph2["Quality Gates"]
    Checkout["actions/checkout@v4"]
    Setup["dtolnay/rust-toolchain@nightly"]
    Format["cargo fmt --all -- --check"]
    Clippy["cargo clippy --target TARGET --all-features"]
    Build["cargo build --target TARGET --all-features"]
    Test["cargo test --target TARGET -- --nocapture"]
end
subgraph subGraph1["CI Job Matrix"]
    Toolchain["Rust Toolchain: nightly"]
    Target1["x86_64-unknown-linux-gnu"]
    Target2["x86_64-unknown-none"]
    Target3["riscv64gc-unknown-none-elf"]
    Target4["aarch64-unknown-none-softfloat"]
end
subgraph subGraph0["Trigger Events"]
    PushEvent["git push"]
    PREvent["Pull Request"]
end

Build --> Test
Checkout --> Setup
Clippy --> Build
DocBuild --> DocDeploy
DocDeploy --> GHPages
Format --> Clippy
PREvent --> Toolchain
PushEvent --> Toolchain
Setup --> Format
Target1 --> Checkout
Target1 --> DocBuild
Target2 --> Checkout
Target3 --> Checkout
Target4 --> Checkout
Toolchain --> Target1
Toolchain --> Target2
Toolchain --> Target3
Toolchain --> Target4
```

### Pipeline Configuration Details

|Stage|Command|Target Filter|Purpose|
| --- | --- | --- | --- |
|Format Check|cargo fmt --all -- --check|All targets|Code style consistency|
|Linting|cargo clippy --target TARGET --all-features|All targets|Static analysis|
|Build|cargo build --target TARGET --all-features|All targets|Compilation verification|
|Testing|cargo test --target TARGET -- --nocapture|Host only|Unit test execution|
|Documentation|cargo doc --no-deps --all-features|Default|API documentation|

Sources: [.github/workflows/ci.yml(L5 - L31)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml#L5-L31) [.github/workflows/ci.yml(L32 - L55)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml#L32-L55)

## Code Quality Standards

The project enforces strict code quality standards through automated tooling and custom configuration.

### Linting Configuration

The clippy configuration includes specific allowances for library design patterns:

```
cargo clippy --target TARGET --all-features -- -A clippy::new_without_default
```

This configuration allows constructors that don't implement `Default`, which is appropriate for specialized data structures like linked lists.

### Documentation Standards

Documentation generation includes strict validation:

```
RUSTDOCFLAGS="-D rustdoc::broken_intra_doc_links -D missing-docs"
```

This configuration:

* Treats broken documentation links as compilation errors (`-D rustdoc::broken_intra_doc_links`)
* Requires documentation for all public APIs (`-D missing-docs`)

### Documentation Deployment

```mermaid
flowchart TD
subgraph subGraph1["Deployment Process"]
    BranchCheck["Check: refs/heads/default_branch"]
    SingleCommit["single-commit: true"]
    GHPagesBranch["branch: gh-pages"]
    TargetDoc["folder: target/doc"]
end
subgraph subGraph0["Documentation Build"]
    CargoDoc["cargo doc --no-deps --all-features"]
    IndexGen["Generate index.html redirect"]
    TreeCmd["cargo tree | head -1 | cut -d' ' -f1"]
end

BranchCheck --> SingleCommit
CargoDoc --> IndexGen
GHPagesBranch --> TargetDoc
IndexGen --> BranchCheck
SingleCommit --> GHPagesBranch
TreeCmd --> IndexGen
```

Sources: [.github/workflows/ci.yml(L25)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml#L25-L25) [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml#L40-L40) [.github/workflows/ci.yml(L44 - L55)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml#L44-L55)

## Contributing Workflow

### Development Environment Setup

1. **Install Rust Nightly Toolchain**

```
rustup toolchain install nightly
rustup component add rust-src clippy rustfmt
```
2. **Add Target Architectures**

```
rustup target add x86_64-unknown-none
rustup target add riscv64gc-unknown-none-elf
rustup target add aarch64-unknown-none-softfloat
```
3. **Verify Installation**

```
rustc --version --verbose
```

### Pre-commit Validation

Before submitting changes, run the complete validation suite locally:

```markdown
# Format check
cargo fmt --all -- --check

# Linting
cargo clippy --all-features -- -A clippy::new_without_default

# Cross-compilation verification
cargo build --target x86_64-unknown-none --all-features
cargo build --target riscv64gc-unknown-none-elf --all-features
cargo build --target aarch64-unknown-none-softfloat --all-features

# Unit tests
cargo test -- --nocapture

# Documentation build
cargo doc --no-deps --all-features
```

This local validation mirrors the CI pipeline and helps catch issues before pushing to the repository.

Sources: [.github/workflows/ci.yml(L20 - L30)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml#L20-L30) [.github/workflows/ci.yml(L44 - L47)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/.github/workflows/ci.yml#L44-L47)