# CI/CD Pipeline

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml)

This document describes the automated continuous integration and deployment pipeline for the arm_pl011 crate. The pipeline handles code quality checks, multi-target compilation, testing, documentation generation, and automated deployment to GitHub Pages.

For information about local development and testing procedures, see [Building and Testing](/arceos-org/arm_pl011/4.1-building-and-testing). For API documentation specifics, see [API Reference](/arceos-org/arm_pl011/3-api-reference).

## Pipeline Overview

The CI/CD pipeline is implemented using GitHub Actions and consists of two primary workflows that execute on every push and pull request to ensure code quality and maintain up-to-date documentation.

### Workflow Architecture

```mermaid
flowchart TD
Trigger["GitHub Events"]
Event1["push"]
Event2["pull_request"]
Pipeline["CI Workflow"]
Job1["ci Job"]
Job2["doc Job"]
Matrix["Matrix Strategy"]
Target1["x86_64-unknown-linux-gnu"]
Target2["x86_64-unknown-none"]
Target3["riscv64gc-unknown-none-elf"]
Target4["aarch64-unknown-none-softfloat"]
DocBuild["cargo doc"]
Deploy["GitHub Pages Deploy"]
QualityChecks["Quality Checks"]
Format["cargo fmt"]
Lint["cargo clippy"]
Build["cargo build"]
Test["cargo test"]

Event1 --> Pipeline
Event2 --> Pipeline
Job1 --> Matrix
Job2 --> Deploy
Job2 --> DocBuild
Matrix --> Target1
Matrix --> Target2
Matrix --> Target3
Matrix --> Target4
Pipeline --> Job1
Pipeline --> Job2
QualityChecks --> Build
QualityChecks --> Format
QualityChecks --> Lint
QualityChecks --> Test
Target1 --> QualityChecks
Target2 --> QualityChecks
Target3 --> QualityChecks
Target4 --> QualityChecks
Trigger --> Event1
Trigger --> Event2
```

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L1-L56)

## CI Job Implementation

The `ci` job implements comprehensive quality assurance through a matrix build strategy that validates the crate across multiple target architectures.

### Matrix Configuration

|Component|Value|
| --- | --- |
|Runner|ubuntu-latest|
|Rust Toolchain|nightly|
|Fail Fast|false|
|Target Count|4 architectures|

The matrix strategy ensures the crate builds correctly across all supported embedded platforms:

```mermaid
flowchart TD
subgraph subGraph2["Toolchain Components"]
    RustSrc["rust-src"]
    Clippy["clippy"]
    Rustfmt["rustfmt"]
end
subgraph subGraph1["Target Matrix"]
    T1["x86_64-unknown-linux-gnu"]
    T2["x86_64-unknown-none"]
    T3["riscv64gc-unknown-none-elf"]
    T4["aarch64-unknown-none-softfloat"]
end
subgraph subGraph0["Matrix Configuration"]
    Toolchain["rust-toolchain: nightly"]
    Strategy["fail-fast: false"]
end

Strategy --> T1
Strategy --> T2
Strategy --> T3
Strategy --> T4
Toolchain --> Clippy
Toolchain --> RustSrc
Toolchain --> Rustfmt
```

Sources: [.github/workflows/ci.yml(L8 - L19)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L8-L19)

### Quality Check Steps

The pipeline implements a four-stage quality verification process:

```mermaid
flowchart TD
Setup["Checkout & Rust Setup"]
Version["rustc --version --verbose"]
Format["cargo fmt --all -- --check"]
Clippy["cargo clippy --target TARGET --all-features"]
Build["cargo build --target TARGET --all-features"]
Test["cargo test --target TARGET"]
Condition["TARGET == x86_64-unknown-linux-gnu?"]
Execute["Execute Tests"]
Skip["Skip Tests"]

Build --> Test
Clippy --> Build
Condition --> Execute
Condition --> Skip
Format --> Clippy
Setup --> Version
Test --> Condition
Version --> Format
```

#### Code Formatting

The `cargo fmt --all -- --check` command validates that all code adheres to standard Rust formatting conventions without making modifications.

#### Linting Analysis

Clippy performs static analysis with the configuration `cargo clippy --target ${{ matrix.targets }} --all-features -- -A clippy::new_without_default`, specifically allowing the `new_without_default` lint for the crate's design patterns.

#### Multi-Target Compilation

Each target architecture undergoes full compilation with `cargo build --target ${{ matrix.targets }} --all-features` to ensure cross-platform compatibility.

#### Unit Testing

Unit tests execute only on the `x86_64-unknown-linux-gnu` target using `cargo test --target ${{ matrix.targets }} -- --nocapture` for comprehensive output visibility.

Sources: [.github/workflows/ci.yml(L20 - L30)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L20-L30)

## Documentation Job

The `doc` job generates and deploys API documentation with strict quality enforcement.

### Documentation Build Process

```mermaid
flowchart TD
DocJob["doc Job"]
Permissions["contents: write"]
EnvCheck["Branch Check"]
DocFlags["RUSTDOCFLAGS Setup"]
Flag1["-D rustdoc::broken_intra_doc_links"]
Flag2["-D missing-docs"]
Build["cargo doc --no-deps --all-features"]
IndexGen["Generate index.html redirect"]
BranchCheck["github.ref == default-branch?"]
Deploy["JamesIves/github-pages-deploy-action@v4"]
Skip["Skip Deployment"]
Target["gh-pages branch"]
Folder["target/doc"]

BranchCheck --> Deploy
BranchCheck --> Skip
Build --> IndexGen
Deploy --> Folder
Deploy --> Target
DocFlags --> Flag1
DocFlags --> Flag2
DocJob --> Permissions
EnvCheck --> DocFlags
Flag1 --> Build
Flag2 --> Build
IndexGen --> BranchCheck
Permissions --> EnvCheck
```

### Documentation Configuration

The documentation build enforces strict quality standards through `RUSTDOCFLAGS`:

|Flag|Purpose|
| --- | --- |
|-D rustdoc::broken_intra_doc_links|Fail on broken internal documentation links|
|-D missing-docs|Require documentation for all public items|

The build process includes automatic index generation:

```
printf '<meta http-equiv="refresh" content="0;url=%s/index.html">' $(cargo tree | head -1 | cut -d' ' -f1) > target/doc/index.html
```

Sources: [.github/workflows/ci.yml(L32 - L48)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L32-L48)

## Deployment Strategy

### GitHub Pages Integration

The deployment uses the `JamesIves/github-pages-deploy-action@v4` action with specific configuration:

```mermaid
flowchart TD
DefaultBranch["Default Branch Push"]
DeployAction["github-pages-deploy-action@v4"]
Config1["single-commit: true"]
Config2["branch: gh-pages"]
Config3["folder: target/doc"]
Result["Documentation Site"]

Config1 --> Result
Config2 --> Result
Config3 --> Result
DefaultBranch --> DeployAction
DeployAction --> Config1
DeployAction --> Config2
DeployAction --> Config3
```

### Conditional Deployment Logic

Documentation deployment occurs only when:

1. The push targets the repository's default branch (`github.ref == env.default-branch`)
2. The documentation build succeeds without errors

For non-default branches and pull requests, the documentation build continues with `continue-on-error: true` to provide feedback without blocking the pipeline.

Sources: [.github/workflows/ci.yml(L38 - L55)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L38-L55)

## Target Architecture Matrix

The pipeline validates compilation across four distinct target architectures, ensuring broad embedded systems compatibility:

### Architecture Support Matrix

|Target|Environment|Use Case|
| --- | --- | --- |
|x86_64-unknown-linux-gnu|Linux userspace|Development and testing|
|x86_64-unknown-none|Bare metal x86|OS kernels and bootloaders|
|riscv64gc-unknown-none-elf|RISC-V embedded|RISC-V based systems|
|aarch64-unknown-none-softfloat|ARM64 embedded|ARM-based embedded systems|

### Build Validation Flow

```mermaid
flowchart TD
Source["Source Code"]
Validation["Multi-Target Validation"]
Linux["x86_64-unknown-linux-gnu"]
BareX86["x86_64-unknown-none"]
RISCV["riscv64gc-unknown-none-elf"]
ARM64["aarch64-unknown-none-softfloat"]
TestSuite["Unit Test Suite"]
CompileOnly["Compile Only"]
Success["Pipeline Success"]

ARM64 --> CompileOnly
BareX86 --> CompileOnly
CompileOnly --> Success
Linux --> TestSuite
RISCV --> CompileOnly
Source --> Validation
TestSuite --> Success
Validation --> ARM64
Validation --> BareX86
Validation --> Linux
Validation --> RISCV
```

Sources: [.github/workflows/ci.yml(L12 - L30)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L12-L30)