# CI/CD Pipeline

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/.github/workflows/ci.yml)

This document describes the automated continuous integration and continuous deployment (CI/CD) pipeline for the arm_gicv2 crate. The pipeline is implemented using GitHub Actions and handles code quality assurance, multi-target builds, testing, and automated documentation deployment.

For information about the build system configuration and dependencies, see [Build System and Dependencies](/arceos-org/arm_gicv2/4.1-build-system-and-dependencies). For development environment setup, see [Development Environment](/arceos-org/arm_gicv2/4.3-development-environment).

## Pipeline Overview

The CI/CD pipeline consists of two primary workflows that execute on every push and pull request, ensuring code quality and maintaining up-to-date documentation.

```mermaid
flowchart TD
subgraph subGraph3["Doc Job Outputs"]
    DocBuild["API documentation"]
    Deploy["GitHub Pages deployment"]
end
subgraph subGraph2["CI Job Outputs"]
    Build["Multi-target builds"]
    Tests["Unit tests"]
    Quality["Code quality checks"]
end
subgraph subGraph1["GitHub Actions Workflows"]
    CI["ci job"]
    DOC["doc job"]
end
subgraph subGraph0["Trigger Events"]
    Push["push"]
    PR["pull_request"]
end

CI --> Build
CI --> Quality
CI --> Tests
DOC --> Deploy
DOC --> DocBuild
PR --> CI
PR --> DOC
Push --> CI
Push --> DOC
```

**CI/CD Pipeline Overview**

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/.github/workflows/ci.yml#L1-L56)

## CI Job Configuration

The main `ci` job runs quality assurance checks and builds across multiple target architectures using a matrix strategy.

### Matrix Strategy

The pipeline uses a fail-fast matrix configuration to test across multiple target platforms:

|Target Architecture|Purpose|
| --- | --- |
|x86_64-unknown-linux-gnu|Standard Linux development and testing|
|x86_64-unknown-none|Bare-metal x86_64 environments|
|riscv64gc-unknown-none-elf|RISC-V embedded systems|
|aarch64-unknown-none-softfloat|ARM64 embedded systems|

```mermaid
flowchart TD
subgraph subGraph2["CI Steps for Each Target"]
    Setup["actions/checkout@v4dtolnay/rust-toolchain@nightly"]
    Format["cargo fmt --all -- --check"]
    Lint["cargo clippy --target TARGET --all-features"]
    Build["cargo build --target TARGET --all-features"]
    Test["cargo test --target TARGET"]
end
subgraph subGraph1["Matrix Strategy"]
    Toolchain["nightly toolchain"]
    subgraph subGraph0["Target Architectures"]
        X86Linux["x86_64-unknown-linux-gnu"]
        X86None["x86_64-unknown-none"]
        RISCV["riscv64gc-unknown-none-elf"]
        ARM64["aarch64-unknown-none-softfloat"]
    end
end

ARM64 --> Setup
Build --> Test
Format --> Lint
Lint --> Build
RISCV --> Setup
Setup --> Format
Toolchain --> ARM64
Toolchain --> RISCV
Toolchain --> X86Linux
Toolchain --> X86None
X86Linux --> Setup
X86None --> Setup
```

**CI Job Matrix and Execution Flow**

Sources: [.github/workflows/ci.yml(L6 - L30)&emsp;](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/.github/workflows/ci.yml#L6-L30)

### Quality Assurance Steps

The CI pipeline enforces code quality through multiple automated checks:

1. **Code Formatting**: Uses `cargo fmt` with the `--check` flag to ensure consistent code formatting
2. **Linting**: Runs `cargo clippy` with all features enabled, specifically allowing the `clippy::new_without_default` lint
3. **Building**: Compiles the crate for each target architecture with all features enabled
4. **Testing**: Executes unit tests, but only for the `x86_64-unknown-linux-gnu` target

```mermaid
flowchart TD
subgraph subGraph1["Failure Conditions"]
    FormatFail["Format violations"]
    ClippyFail["Lint violations"]
    BuildFail["Compilation errors"]
    TestFail["Test failures"]
end
subgraph subGraph0["Quality Gates"]
    FormatCheck["cargo fmt --all -- --check"]
    ClippyCheck["cargo clippy --target TARGET --all-features-- -A clippy::new_without_default"]
    BuildCheck["cargo build --target TARGET --all-features"]
    TestCheck["cargo test --target TARGET(Linux only)"]
end

BuildCheck --> BuildFail
BuildCheck --> TestCheck
ClippyCheck --> BuildCheck
ClippyCheck --> ClippyFail
FormatCheck --> ClippyCheck
FormatCheck --> FormatFail
TestCheck --> TestFail
```

**Quality Assurance Gate Sequence**

Sources: [.github/workflows/ci.yml(L22 - L30)&emsp;](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/.github/workflows/ci.yml#L22-L30)

## Documentation Job

The `doc` job handles API documentation generation and deployment to GitHub Pages.

### Documentation Build Process

The documentation build process includes strict documentation standards and automated deployment:

```mermaid
flowchart TD
subgraph subGraph2["Deployment Logic"]
    BranchCheck["Check if default branch"]
    Deploy["JamesIves/github-pages-deploy-action@v4"]
    GHPages["Deploy to gh-pages branch"]
end
subgraph subGraph1["Build Process"]
    DocBuild["cargo doc --no-deps --all-features"]
    IndexGen["Generate index.html redirect"]
    TreeExtract["Extract crate name from cargo tree"]
end
subgraph subGraph0["Documentation Environment"]
    DocEnv["RUSTDOCFLAGS:-D rustdoc::broken_intra_doc_links-D missing-docs"]
    Checkout["actions/checkout@v4"]
    Toolchain["dtolnay/rust-toolchain@nightly"]
end

BranchCheck --> Deploy
Checkout --> Toolchain
Deploy --> GHPages
DocBuild --> IndexGen
DocEnv --> Checkout
IndexGen --> TreeExtract
Toolchain --> DocBuild
TreeExtract --> BranchCheck
```

**Documentation Build and Deployment Flow**

Sources: [.github/workflows/ci.yml(L32 - L56)&emsp;](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/.github/workflows/ci.yml#L32-L56)

### Documentation Standards

The pipeline enforces strict documentation requirements through `RUSTDOCFLAGS`:

* **Broken Links**: `-D rustdoc::broken_intra_doc_links` treats broken documentation links as errors
* **Missing Documentation**: `-D missing-docs` requires all public items to have documentation

### Deployment Configuration

Documentation deployment occurs automatically under specific conditions:

* **Branch Condition**: Only deploys when the push is to the default branch
* **Target Branch**: Deploys to the `gh-pages` branch
* **Single Commit**: Uses `single-commit: true` to maintain a clean deployment history
* **Source Folder**: Deploys the `target/doc` directory containing generated documentation

Sources: [.github/workflows/ci.yml(L38 - L55)&emsp;](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/.github/workflows/ci.yml#L38-L55)

## Toolchain and Component Configuration

The pipeline uses the Rust nightly toolchain with specific components required for the complete CI/CD process:

|Component|Purpose|
| --- | --- |
|rust-src|Source code for cross-compilation|
|clippy|Linting and code analysis|
|rustfmt|Code formatting|

The toolchain setup includes automatic target installation for all matrix targets, ensuring consistent build environments across all supported architectures.

Sources: [.github/workflows/ci.yml(L15 - L19)&emsp;](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/.github/workflows/ci.yml#L15-L19)