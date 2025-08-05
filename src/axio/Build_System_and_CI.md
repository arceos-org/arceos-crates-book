# Build System and CI

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml)
> * [Cargo.toml](https://github.com/arceos-org/axio/blob/a675e6d5/Cargo.toml)

This document covers the build system configuration and continuous integration pipeline for the axio crate. It explains how the project is structured for compilation across multiple target platforms, the automated testing strategy, and the documentation deployment process. For information about the crate's feature configuration and dependencies, see [Crate Configuration and Features](/arceos-org/axio/3-crate-configuration-and-features).

## Build Configuration

The axio crate uses a minimal build configuration designed for `no_std` compatibility across diverse target platforms. The build system is defined primarily through `Cargo.toml` and supports conditional compilation based on feature flags.

### Package Metadata and Dependencies

The crate is configured as a library package with specific metadata targeting the embedded and OS kernel development ecosystem:

|Configuration|Value|
| --- | --- |
|Package Name|axio|
|Version|0.1.1|
|Edition|2021|
|License|Dual/Triple licensed (GPL-3.0-or-later OR Apache-2.0 OR MulanPSL-2.0)|

The dependency structure is intentionally minimal, with only one required dependency:

* `axerrno = "0.1"` - Provides error handling types compatible with `no_std` environments

### Feature Gate Configuration

The build system supports feature-based conditional compilation through two defined features:

```mermaid
flowchart TD
subgraph subGraph2["Available APIs"]
    CORE_API["Core I/O Traitsread(), write(), seek()"]
    ALLOC_API["Dynamic Memory APIsread_to_end(), read_to_string()"]
end
subgraph subGraph1["Compilation Modes"]
    MINIMAL["Minimal Buildno_std only"]
    ENHANCED["Enhanced Buildno_std + alloc"]
end
subgraph subGraph0["Feature Configuration"]
    DEFAULT["default = []"]
    ALLOC["alloc = []"]
end

ALLOC --> ENHANCED
DEFAULT --> MINIMAL
ENHANCED --> ALLOC_API
ENHANCED --> CORE_API
MINIMAL --> CORE_API
```

Sources: [Cargo.toml(L14 - L16)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/Cargo.toml#L14-L16)

## CI Pipeline Architecture

The continuous integration system uses GitHub Actions to validate code quality, build compatibility, and documentation generation across multiple target platforms. The pipeline is defined in a single workflow file that orchestrates multiple jobs.

### Workflow Trigger Configuration

The CI pipeline activates on two primary events:

* Push events to any branch
* Pull request events

### Multi-Target Build Matrix

The CI system employs a matrix strategy to test compilation across diverse target platforms:

```mermaid
flowchart TD
subgraph subGraph2["CI Steps"]
    CHECKOUT["actions/checkout@v4"]
    TOOLCHAIN_SETUP["dtolnay/rust-toolchain@nightly"]
    FORMAT_CHECK["cargo fmt --check"]
    CLIPPY_CHECK["cargo clippy"]
    BUILD_STEP["cargo build"]
    TEST_STEP["cargo test"]
end
subgraph subGraph1["Build Matrix"]
    TOOLCHAIN["nightly toolchain"]
    TARGET1["x86_64-unknown-linux-gnu"]
    TARGET2["x86_64-unknown-none"]
    TARGET3["riscv64gc-unknown-none-elf"]
    TARGET4["aarch64-unknown-none-softfloat"]
end
subgraph subGraph0["CI Workflow"]
    TRIGGER["push/pull_request events"]
    CI_JOB["ci job"]
    DOC_JOB["doc job"]
end

BUILD_STEP --> TEST_STEP
CHECKOUT --> TOOLCHAIN_SETUP
CI_JOB --> CHECKOUT
CI_JOB --> TOOLCHAIN
CLIPPY_CHECK --> BUILD_STEP
FORMAT_CHECK --> CLIPPY_CHECK
TOOLCHAIN --> TARGET1
TOOLCHAIN --> TARGET2
TOOLCHAIN --> TARGET3
TOOLCHAIN --> TARGET4
TOOLCHAIN_SETUP --> FORMAT_CHECK
TRIGGER --> CI_JOB
TRIGGER --> DOC_JOB
```

Sources: [.github/workflows/ci.yml(L1 - L31)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L1-L31)

### Target Platform Categories

The build matrix validates compatibility across three categories of target platforms:

|Target|Architecture|Environment|Purpose|
| --- | --- | --- | --- |
|x86_64-unknown-linux-gnu|x86_64|Linux with std|Testing and validation|
|x86_64-unknown-none|x86_64|Bare metal|OS kernel development|
|riscv64gc-unknown-none-elf|RISC-V|Bare metal|Embedded systems|
|aarch64-unknown-none-softfloat|ARM64|Bare metal|ARM-based embedded|

Sources: [.github/workflows/ci.yml(L12)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L12-L12)

## Quality Assurance Pipeline

The CI system implements a comprehensive quality assurance strategy that validates code formatting, linting, compilation, and functional correctness.

### Code Quality Checks

The quality assurance pipeline executes the following checks in sequence:

```mermaid
flowchart TD
START["CI Job Start"]
VERSION_CHECK["rustc --version --verbose"]
FORMAT["cargo fmt --all -- --check"]
CLIPPY["cargo clippy --target TARGET --all-features"]
BUILD["cargo build --target TARGET --all-features"]
TEST_CONDITION["target == x86_64-unknown-linux-gnu?"]
UNIT_TEST["cargo test --target TARGET -- --nocapture"]
END["Job Complete"]
CLIPPY_CONFIG["clippy config:-A clippy::new_without_default"]

BUILD --> TEST_CONDITION
CLIPPY --> BUILD
CLIPPY --> CLIPPY_CONFIG
FORMAT --> CLIPPY
START --> VERSION_CHECK
TEST_CONDITION --> END
TEST_CONDITION --> UNIT_TEST
UNIT_TEST --> END
VERSION_CHECK --> FORMAT
```

Sources: [.github/workflows/ci.yml(L20 - L30)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L20-L30)

### Testing Strategy

Unit tests are executed only on the `x86_64-unknown-linux-gnu` target, which provides a standard library environment suitable for test execution. The testing configuration includes:

* **Test Command**: `cargo test --target x86_64-unknown-linux-gnu -- --nocapture`
* **Output Mode**: No capture mode for detailed test output
* **Conditional Execution**: Only runs on Linux GNU target to avoid `no_std` test environment complications

Sources: [.github/workflows/ci.yml(L28 - L30)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L28-L30)

## Documentation Generation and Deployment

The CI pipeline includes a separate job dedicated to documentation generation and automated deployment to GitHub Pages.

### Documentation Build Process

```mermaid
flowchart TD
subgraph subGraph2["Deploy Configuration"]
    BRANCH["gh-pages branch"]
    FOLDER["target/doc"]
    COMMIT_MODE["single-commit: true"]
end
subgraph subGraph1["Build Configuration"]
    RUSTDOCFLAGS["-D rustdoc::broken_intra_doc_links-D missing-docs"]
    PERMISSIONS["contents: write"]
end
subgraph subGraph0["Documentation Job"]
    DOC_TRIGGER["push/PR events"]
    DOC_CHECKOUT["actions/checkout@v4"]
    DOC_TOOLCHAIN["dtolnay/rust-toolchain@nightly"]
    DOC_BUILD["cargo doc --no-deps --all-features"]
    INDEX_GEN["Generate index.html redirect"]
    DEPLOY_CHECK["ref == default-branch?"]
    DEPLOY["JamesIves/github-pages-deploy-action@v4"]
end

DEPLOY --> BRANCH
DEPLOY --> COMMIT_MODE
DEPLOY --> FOLDER
DEPLOY_CHECK --> DEPLOY
DOC_BUILD --> INDEX_GEN
DOC_CHECKOUT --> DOC_TOOLCHAIN
DOC_TOOLCHAIN --> DOC_BUILD
DOC_TRIGGER --> DOC_CHECKOUT
INDEX_GEN --> DEPLOY_CHECK
PERMISSIONS --> DEPLOY
RUSTDOCFLAGS --> DOC_BUILD
```

### Documentation Quality Enforcement

The documentation build process enforces strict quality standards through `RUSTDOCFLAGS`:

* **Broken Link Detection**: `-D rustdoc::broken_intra_doc_links` treats broken documentation links as errors
* **Missing Documentation**: `-D missing-docs` requires documentation for all public APIs

The index generation creates a redirect page using the crate name extracted from `cargo tree` output, providing seamless navigation to the main documentation.

Sources: [.github/workflows/ci.yml(L32 - L55)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L32-L55)

### Deployment Strategy

Documentation deployment follows a conditional strategy:

* **Automatic Deployment**: Occurs only on pushes to the default branch
* **Target Branch**: `gh-pages` branch for GitHub Pages hosting
* **Deployment Mode**: Single commit to maintain clean history
* **Content Source**: `target/doc` directory containing generated documentation

Sources: [.github/workflows/ci.yml(L49 - L55)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L49-L55)