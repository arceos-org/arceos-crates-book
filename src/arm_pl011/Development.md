# Development

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.gitignore)
> * [Cargo.toml](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/Cargo.toml)

This section provides a comprehensive guide for contributors to the `arm_pl011` crate, covering the development workflow, multi-target building, code quality standards, and continuous integration processes. The material focuses on the practical aspects of contributing to this embedded systems driver library.

For detailed API documentation and usage patterns, see [API Reference](/arceos-org/arm_pl011/3-api-reference). For hardware-specific implementation details, see [Core Implementation](/arceos-org/arm_pl011/2-core-implementation).

## Development Environment Setup

The `arm_pl011` crate is designed as a `no_std` embedded library that supports multiple target architectures. Development requires the Rust nightly toolchain with specific components and target platforms.

### Required Toolchain Components

The project uses Rust nightly with the following components as defined in the CI configuration:

|Component|Purpose|
| --- | --- |
|rust-src|Source code for cross-compilation|
|clippy|Linting and code quality checks|
|rustfmt|Code formatting enforcement|

### Supported Target Platforms

The crate maintains compatibility across four distinct target architectures:

|Target|Use Case|
| --- | --- |
|x86_64-unknown-linux-gnu|Development and testing on Linux|
|x86_64-unknown-none|Bare metal x86_64 systems|
|riscv64gc-unknown-none-elf|RISC-V embedded systems|
|aarch64-unknown-none-softfloat|ARM64 embedded systems without FPU|

Sources: [.github/workflows/ci.yml(L11 - L12)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L11-L12) [.github/workflows/ci.yml(L19)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L19-L19)

## Multi-Target Development Workflow

The development process accommodates the diverse embedded systems ecosystem by supporting multiple target architectures through a unified workflow.

### Target Architecture Matrix

```mermaid
flowchart TD
subgraph subGraph2["Build Verification"]
    FMT["cargo fmt --check"]
    CLIPPY["cargo clippy"]
    BUILD["cargo build"]
    TEST["cargo test(Linux only)"]
end
subgraph subGraph1["Target Matrix"]
    LINUX["x86_64-unknown-linux-gnuDevelopment & Testing"]
    BARE_X86["x86_64-unknown-noneBare Metal x86"]
    RISCV["riscv64gc-unknown-none-elfRISC-V Systems"]
    ARM64["aarch64-unknown-none-softfloatARM64 Embedded"]
end
subgraph subGraph0["Development Workflow"]
    SRC["Source Codesrc/lib.rssrc/pl011.rs"]
    TOOLCHAIN["Rust Nightlyrust-src, clippy, rustfmt"]
end

ARM64 --> FMT
BARE_X86 --> FMT
BUILD --> TEST
CLIPPY --> BUILD
FMT --> CLIPPY
LINUX --> FMT
RISCV --> FMT
SRC --> TOOLCHAIN
TOOLCHAIN --> ARM64
TOOLCHAIN --> BARE_X86
TOOLCHAIN --> LINUX
TOOLCHAIN --> RISCV
```

Sources: [.github/workflows/ci.yml(L11 - L12)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L11-L12) [.github/workflows/ci.yml(L23 - L30)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L23-L30)

### Local Development Commands

For local development, contributors should verify their changes against all supported targets:

```markdown
# Format check
cargo fmt --all -- --check

# Linting for each target
cargo clippy --target <TARGET> --all-features

# Build verification
cargo build --target <TARGET> --all-features

# Unit tests (Linux only)
cargo test --target x86_64-unknown-linux-gnu -- --nocapture
```

The `--all-features` flag ensures compatibility with the complete feature set defined in [`Cargo.toml(L15)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/`Cargo.toml#L15-L15)

Sources: [.github/workflows/ci.yml(L23 - L30)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L23-L30)

## Continuous Integration Pipeline

The project maintains code quality through an automated CI/CD pipeline that validates all changes across the supported target matrix.

### CI/CD Architecture

```mermaid
flowchart TD
subgraph subGraph4["Documentation Pipeline"]
    DOC_BUILD["cargo doc --no-deps"]
    RUSTDOC_FLAGS["-D rustdoc::broken_intra_doc_links-D missing-docs"]
    GH_PAGES["Deploy to gh-pages(main branch only)"]
end
subgraph subGraph3["Quality Gates"]
    FORMAT["cargo fmt --check"]
    LINT["cargo clippy"]
    BUILD_ALL["cargo build(all targets)"]
    UNIT_TEST["cargo test(Linux only)"]
end
subgraph subGraph2["CI Matrix Strategy"]
    TOOLCHAIN["rust-toolchain: nightly"]
    TARGET_MATRIX["targets matrix:x86_64-unknown-linux-gnux86_64-unknown-noneriscv64gc-unknown-none-elfaarch64-unknown-none-softfloat"]
end
subgraph subGraph1["CI Jobs"]
    CI_JOB["ci jobubuntu-latest"]
    DOC_JOB["doc jobubuntu-latest"]
end
subgraph subGraph0["Trigger Events"]
    PUSH["git push"]
    PR["pull_request"]
end

BUILD_ALL --> UNIT_TEST
CI_JOB --> TARGET_MATRIX
CI_JOB --> TOOLCHAIN
DOC_BUILD --> RUSTDOC_FLAGS
DOC_JOB --> DOC_BUILD
FORMAT --> LINT
LINT --> BUILD_ALL
PR --> CI_JOB
PR --> DOC_JOB
PUSH --> CI_JOB
PUSH --> DOC_JOB
RUSTDOC_FLAGS --> GH_PAGES
TARGET_MATRIX --> FORMAT
```

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L1-L56)

### Quality Enforcement Standards

The CI pipeline enforces strict quality standards through multiple validation stages:

#### Code Formatting

All code must pass `rustfmt` validation with the project's formatting rules. The CI fails if formatting inconsistencies are detected.

#### Linting Rules

The project uses `clippy` with custom configuration that allows the `clippy::new_without_default` warning while maintaining strict standards for other potential issues.

#### Documentation Requirements

Documentation builds enforce strict standards through `RUSTDOCFLAGS`:

* `-D rustdoc::broken_intra_doc_links`: Fails on broken documentation links
* `-D missing-docs`: Requires documentation for all public interfaces

Sources: [.github/workflows/ci.yml(L23 - L25)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L23-L25) [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L40-L40)

### Testing Strategy

The project implements a targeted testing approach:

* **Unit Tests**: Execute only on `x86_64-unknown-linux-gnu` for practical development iteration
* **Build Verification**: All targets must compile successfully to ensure cross-platform compatibility
* **Documentation Tests**: Included in the documentation build process

This strategy balances comprehensive validation with CI resource efficiency, since the core functionality is hardware-agnostic while the compilation targets verify platform compatibility.

Sources: [.github/workflows/ci.yml(L28 - L30)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L28-L30)

### Documentation Deployment

The documentation pipeline automatically generates and deploys API documentation to GitHub Pages for the main branch. The process includes:

1. Building documentation with `cargo doc --no-deps --all-features`
2. Generating an index redirect page based on the crate name from `cargo tree`
3. Deploying to the `gh-pages` branch using single-commit strategy

This ensures that the latest documentation is always available at the project's GitHub Pages URL as specified in [`Cargo.toml(L10)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/`Cargo.toml#L10-L10)

Sources: [.github/workflows/ci.yml(L44 - L55)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/.github/workflows/ci.yml#L44-L55)