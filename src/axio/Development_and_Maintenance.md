# Development and Maintenance

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-org/axio/blob/a675e6d5/.gitignore)

This document provides guidance for contributors and maintainers of the `axio` crate, covering development workflows, build processes, and quality assurance practices. It outlines the automated systems that ensure code quality and compatibility across multiple target environments.

For detailed information about the CI pipeline and multi-target builds, see [Build System and CI](/arceos-org/axio/6.1-build-system-and-ci). For development environment setup and configuration files, see [Project Configuration](/arceos-org/axio/6.2-project-configuration).

## Development Workflow Overview

The `axio` crate follows a rigorous development process designed to maintain compatibility across diverse `no_std` environments. The development workflow emphasizes automated quality checks, multi-target validation, and comprehensive documentation.

### Multi-Target Development Strategy

The crate targets multiple architectures and environments simultaneously, requiring careful consideration of platform-specific constraints:

```mermaid
flowchart TD
DEV["Developer Changes"]
PR["Pull Request"]
PUSH["Direct Push"]
CI["CI Pipeline"]
SETUP["Toolchain Setup"]
MATRIX["Target Matrix Execution"]
LINUX["x86_64-unknown-linux-gnu(Linux with std)"]
BARE_X86["x86_64-unknown-none(Bare metal x86_64)"]
RISCV["riscv64gc-unknown-none-elf(RISC-V bare metal)"]
ARM["aarch64-unknown-none-softfloat(ARM64 bare metal)"]
CHECKS["Quality Checks"]
FORMAT["cargo fmt --check"]
CLIPPY["cargo clippy"]
BUILD["cargo build"]
TEST["cargo test"]
DOC_JOB["Documentation Job"]
DOC_BUILD["cargo doc"]
DEPLOY["GitHub Pages Deploy"]

ARM --> CHECKS
BARE_X86 --> CHECKS
CHECKS --> BUILD
CHECKS --> CLIPPY
CHECKS --> FORMAT
CHECKS --> TEST
CI --> DOC_JOB
CI --> SETUP
DEV --> PR
DOC_BUILD --> DEPLOY
DOC_JOB --> DOC_BUILD
LINUX --> CHECKS
MATRIX --> ARM
MATRIX --> BARE_X86
MATRIX --> LINUX
MATRIX --> RISCV
PR --> CI
PUSH --> CI
RISCV --> CHECKS
SETUP --> MATRIX
TEST --> LINUX
```

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L1-L56)

### Quality Assurance Pipeline

The development process enforces multiple layers of quality assurance through automated checks:

|Check Type|Tool|Purpose|Scope|
| --- | --- | --- | --- |
|Code Formatting|cargo fmt --check|Ensures consistent code style|All targets|
|Linting|cargo clippy --all-features|Catches common mistakes and improvements|All targets|
|Compilation|cargo build --all-features|Verifies code compiles successfully|All targets|
|Unit Testing|cargo test|Validates functionality|Linux target only|
|Documentation|cargo doc --no-deps --all-features|Ensures documentation builds correctly|All features|

The CI configuration uses specific flags to enforce documentation quality through `RUSTDOCFLAGS: -D rustdoc::broken_intra_doc_links -D missing-docs` [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L40-L40) ensuring all public APIs are properly documented.

## Toolchain Requirements

The project requires the nightly Rust toolchain with specific components and targets:

```mermaid
flowchart TD
NIGHTLY["nightly toolchain"]
COMPONENTS["Required Components"]
RUST_SRC["rust-src(for no_std targets)"]
CLIPPY["clippy(for linting)"]
RUSTFMT["rustfmt(for formatting)"]
TARGETS["Target Platforms"]
STD_TARGET["x86_64-unknown-linux-gnu"]
NOSTD_TARGETS["no_std targets"]
X86_BARE["x86_64-unknown-none"]
RISCV_BARE["riscv64gc-unknown-none-elf"]
ARM_BARE["aarch64-unknown-none-softfloat"]

COMPONENTS --> CLIPPY
COMPONENTS --> RUSTFMT
COMPONENTS --> RUST_SRC
NIGHTLY --> COMPONENTS
NIGHTLY --> TARGETS
NOSTD_TARGETS --> ARM_BARE
NOSTD_TARGETS --> RISCV_BARE
NOSTD_TARGETS --> X86_BARE
TARGETS --> NOSTD_TARGETS
TARGETS --> STD_TARGET
```

Sources: [.github/workflows/ci.yml(L15 - L19)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L15-L19)

## Documentation Generation and Deployment

The crate maintains automatically generated documentation deployed to GitHub Pages. The documentation build process includes:

1. **Strict Documentation Standards**: The build fails on missing documentation or broken internal links
2. **Feature-Complete Documentation**: Built with `--all-features` to include all available functionality
3. **Automatic Deployment**: Documentation is automatically deployed from the default branch
4. **Custom Index**: Generates a redirect index page pointing to the main crate documentation

The documentation deployment uses a single-commit strategy to the `gh-pages` branch, ensuring a clean deployment history [.github/workflows/ci.yml(L53 - L55)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L53-L55)

## Development Environment Setup

### Repository Structure

The development environment excludes certain files and directories from version control:

```mermaid
flowchart TD
REPO["Repository Root"]
TRACKED["Tracked Files"]
IGNORED["Ignored Files"]
SRC["src/ directory"]
CARGO["Cargo.toml"]
README["README.md"]
CI[".github/workflows/"]
TARGET["target/(build artifacts)"]
VSCODE[".vscode/(editor config)"]
DSSTORE[".DS_Store(macOS metadata)"]
LOCK["Cargo.lock(dependency versions)"]

IGNORED --> DSSTORE
IGNORED --> LOCK
IGNORED --> TARGET
IGNORED --> VSCODE
REPO --> IGNORED
REPO --> TRACKED
TRACKED --> CARGO
TRACKED --> CI
TRACKED --> README
TRACKED --> SRC
```

Sources: [.gitignore(L1 - L5)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.gitignore#L1-L5)

### Build Artifact Management

The `target/` directory is excluded from version control as it contains build artifacts that are regenerated during compilation. The `Cargo.lock` file is also ignored, following Rust library conventions where lock files are typically not committed for libraries to allow downstream consumers flexibility in dependency resolution.

## Continuous Integration Architecture

The CI system uses a matrix strategy to validate the crate across multiple target environments simultaneously:

```mermaid
flowchart TD
TRIGGER["CI Triggers"]
EVENTS["Event Types"]
PUSH["push events"]
PR["pull_request events"]
STRATEGY["Matrix Strategy"]
FAIL_FAST["fail-fast: false"]
RUST_VER["rust-toolchain: [nightly]"]
TARGET_MATRIX["Target Matrix"]
T1["x86_64-unknown-linux-gnu"]
T2["x86_64-unknown-none"]
T3["riscv64gc-unknown-none-elf"]
T4["aarch64-unknown-none-softfloat"]
PARALLEL["Parallel Execution"]
JOB1["ci job"]
JOB2["doc job"]
VALIDATION["Code Validation"]
DOC_GEN["Documentation Generation"]

EVENTS --> PR
EVENTS --> PUSH
JOB1 --> VALIDATION
JOB2 --> DOC_GEN
PARALLEL --> JOB1
PARALLEL --> JOB2
STRATEGY --> FAIL_FAST
STRATEGY --> PARALLEL
STRATEGY --> RUST_VER
STRATEGY --> TARGET_MATRIX
TARGET_MATRIX --> T1
TARGET_MATRIX --> T2
TARGET_MATRIX --> T3
TARGET_MATRIX --> T4
TRIGGER --> EVENTS
TRIGGER --> STRATEGY
```

Sources: [.github/workflows/ci.yml(L5 - L12)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L5-L12) [.github/workflows/ci.yml(L32 - L36)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L32-L36)

The `fail-fast: false` configuration ensures that failures in one target don't prevent testing of other targets, providing comprehensive feedback about platform-specific issues [.github/workflows/ci.yml(L9)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L9-L9)

## Contributing Guidelines

### Code Style and Standards

All contributions must pass the automated quality checks:

* **Formatting**: Code must be formatted using `cargo fmt` with default settings
* **Linting**: All `clippy` warnings must be addressed, with the exception of `clippy::new_without_default` which is explicitly allowed [.github/workflows/ci.yml(L25)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L25-L25)
* **Documentation**: All public APIs must be documented to pass the strict documentation checks
* **Testing**: New functionality should include appropriate unit tests

### Testing Strategy

The testing approach recognizes the constraints of different target environments:

* **Unit Tests**: Run only on `x86_64-unknown-linux-gnu` target due to standard library requirements [.github/workflows/ci.yml(L29 - L30)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L29-L30)
* **Compilation Tests**: All targets must compile successfully to ensure `no_std` compatibility
* **Feature Testing**: All tests run with `--all-features` to validate optional functionality

This testing strategy ensures that while functionality is validated thoroughly on one platform, compilation compatibility is verified across all supported targets.

Sources: [.github/workflows/ci.yml(L24 - L30)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/.github/workflows/ci.yml#L24-L30)