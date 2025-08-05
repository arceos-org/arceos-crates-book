# Contributing

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.gitignore)

This document provides guidelines for contributing to the page_table_multiarch project, including development environment setup, code quality standards, testing requirements, and the contribution workflow. For information about building and running tests, see [Building and Testing](/arceos-org/page_table_multiarch/5.1-building-and-testing).

## Development Environment Setup

### Required Tools

The project requires the nightly Rust toolchain with specific components and targets for multi-architecture support.

```mermaid
flowchart TD
subgraph Targets["Targets"]
    LINUX["x86_64-unknown-linux-gnu"]
    BARE["x86_64-unknown-none"]
    RISCV["riscv64gc-unknown-none-elf"]
    ARM["aarch64-unknown-none-softfloat"]
    LOONG["loongarch64-unknown-none-softfloat"]
end
subgraph Components["Components"]
    RUSTSRC["rust-src"]
    CLIPPY["clippy"]
    RUSTFMT["rustfmt"]
end
subgraph subGraph0["Rust Toolchain Requirements"]
    NIGHTLY["nightly toolchain"]
    COMPONENTS["Required Components"]
    TARGETS["Target Architectures"]
end
DEV["Development Environment"]

COMPONENTS --> CLIPPY
COMPONENTS --> RUSTFMT
COMPONENTS --> RUSTSRC
DEV --> NIGHTLY
NIGHTLY --> COMPONENTS
NIGHTLY --> TARGETS
TARGETS --> ARM
TARGETS --> BARE
TARGETS --> LINUX
TARGETS --> LOONG
TARGETS --> RISCV
```

**Development Environment Setup**

Install the required toolchain components as specified in the CI configuration:

Sources: [.github/workflows/ci.yml(L15 - L19)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L15-L19)

### IDE Configuration

The project includes basic IDE configuration exclusions in `.gitignore` for common development environments:

* `/target` - Rust build artifacts
* `/.vscode` - Visual Studio Code settings
* `.DS_Store` - macOS system files

Sources: [.gitignore(L1 - L4)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.gitignore#L1-L4)

## Code Quality Standards

### Formatting Requirements

All code must be formatted using `cargo fmt` with the default Rust formatting rules. The CI pipeline enforces this requirement:

```mermaid
flowchart TD
PR["Pull Request"]
CHECK["Format Check"]
PASS["Check Passes"]
FAIL["Check Fails"]
REJECT["PR Blocked"]
CONTINUE["Continue CI"]

CHECK --> FAIL
CHECK --> PASS
FAIL --> REJECT
PASS --> CONTINUE
PR --> CHECK
```

**Formatting Enforcement**

The CI runs `cargo fmt --all -- --check` to verify formatting compliance.

Sources: [.github/workflows/ci.yml(L22 - L23)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L22-L23)

### Linting Standards

Code must pass Clippy analysis with the project's configured lint rules:

```mermaid
flowchart TD
subgraph subGraph1["Allowed Exceptions"]
    NEW_WITHOUT_DEFAULT["clippy::new_without_default"]
end
subgraph subGraph0["Lint Configuration"]
    ALLOW["Allowed Lints"]
    TARGETS_ALL["All Target Architectures"]
    FEATURES["All Features Enabled"]
end
CLIPPY["Clippy Analysis"]

ALLOW --> NEW_WITHOUT_DEFAULT
CLIPPY --> ALLOW
CLIPPY --> FEATURES
CLIPPY --> TARGETS_ALL
```

**Clippy Configuration**

The project allows the `clippy::new_without_default` lint, indicating that `new()` methods without corresponding `Default` implementations are acceptable in this codebase.

Sources: [.github/workflows/ci.yml(L24 - L25)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L24-L25)

## Testing Requirements

### Test Execution Strategy

The project uses a targeted testing approach where unit tests only run on the `x86_64-unknown-linux-gnu` target, while other targets focus on compilation validation:

```mermaid
flowchart TD
subgraph subGraph1["Test Execution"]
    LINUX_ONLY["x86_64-unknown-linux-gnu only"]
    UNIT_TESTS["cargo test"]
    DOC_CFG["RUSTFLAGS: --cfg doc"]
end
subgraph subGraph0["Build Verification"]
    ALL_TARGETS["All Targets"]
    BUILD["cargo build"]
    CLIPPY_CHECK["cargo clippy"]
end
CI["CI Pipeline"]

ALL_TARGETS --> BUILD
ALL_TARGETS --> CLIPPY_CHECK
CI --> ALL_TARGETS
CI --> LINUX_ONLY
LINUX_ONLY --> DOC_CFG
LINUX_ONLY --> UNIT_TESTS
```

**Test Environment Configuration**

Unit tests run with `RUSTFLAGS: --cfg doc` to enable documentation-conditional code paths.

Sources: [.github/workflows/ci.yml(L28 - L32)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L28-L32)

### Multi-Architecture Validation

While unit tests only run on Linux, the CI ensures that all architecture-specific code compiles correctly across all supported targets:

|Target|Purpose|Validation|
| --- | --- | --- |
|x86_64-unknown-linux-gnu|Development and testing|Full test suite|
|x86_64-unknown-none|Bare metal x86_64|Build + Clippy|
|riscv64gc-unknown-none-elf|RISC-V bare metal|Build + Clippy|
|aarch64-unknown-none-softfloat|ARM64 bare metal|Build + Clippy|
|loongarch64-unknown-none-softfloat|LoongArch64 bare metal|Build + Clippy|

Sources: [.github/workflows/ci.yml(L10 - L12)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L10-L12) [.github/workflows/ci.yml(L24 - L27)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L24-L27)

## Documentation Standards

### Documentation Generation

The project maintains comprehensive API documentation that is automatically built and deployed:

```mermaid
flowchart TD
subgraph Deployment["Deployment"]
    BUILD["cargo doc --no-deps"]
    DEPLOY["GitHub Pages"]
    BRANCH["gh-pages"]
end
subgraph subGraph1["Documentation Flags"]
    INDEX["--enable-index-page"]
    BROKEN_LINKS["-D rustdoc::broken_intra_doc_links"]
    MISSING_DOCS["-D missing-docs"]
end
subgraph subGraph0["Build Configuration"]
    RUSTFLAGS["RUSTFLAGS: --cfg doc"]
    RUSTDOCFLAGS["RUSTDOCFLAGS"]
    ALL_FEATURES["--all-features"]
end
DOC_JOB["Documentation Job"]

BUILD --> DEPLOY
DEPLOY --> BRANCH
DOC_JOB --> ALL_FEATURES
DOC_JOB --> BUILD
DOC_JOB --> RUSTDOCFLAGS
DOC_JOB --> RUSTFLAGS
RUSTDOCFLAGS --> BROKEN_LINKS
RUSTDOCFLAGS --> INDEX
RUSTDOCFLAGS --> MISSING_DOCS
```

**Documentation Requirements**

* All public APIs must have documentation comments
* Documentation links must be valid (enforced by `-D rustdoc::broken_intra_doc_links`)
* Missing documentation is treated as an error (`-D missing-docs`)
* Documentation is built with all features enabled

Sources: [.github/workflows/ci.yml(L40 - L43)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L40-L43) [.github/workflows/ci.yml(L47 - L49)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L47-L49)

## Pull Request Workflow

### CI Pipeline Execution

Every pull request triggers the complete CI pipeline across all supported architectures:

```mermaid
flowchart TD
subgraph subGraph2["Documentation Execution"]
    DOC_BUILD["Build Documentation"]
    DOC_DEPLOY["Deploy to gh-pages"]
end
subgraph subGraph1["CI Matrix Execution"]
    FORMAT["Format Check"]
    CLIPPY_ALL["Clippy (All Targets)"]
    BUILD_ALL["Build (All Targets)"]
    TEST_LINUX["Unit Tests (Linux only)"]
end
subgraph subGraph0["Parallel CI Jobs"]
    CI_MATRIX["CI Matrix Job"]
    DOC_JOB["Documentation Job"]
end
PR["Pull Request"]

CI_MATRIX --> BUILD_ALL
CI_MATRIX --> CLIPPY_ALL
CI_MATRIX --> FORMAT
CI_MATRIX --> TEST_LINUX
DOC_BUILD --> DOC_DEPLOY
DOC_JOB --> DOC_BUILD
PR --> CI_MATRIX
PR --> DOC_JOB
```

**CI Trigger Events**

The CI pipeline runs on both `push` and `pull_request` events, ensuring comprehensive validation.

Sources: [.github/workflows/ci.yml(L3)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L3-L3) [.github/workflows/ci.yml(L5 - L33)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L5-L33) [.github/workflows/ci.yml(L34 - L57)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L34-L57)

### Deployment Process

Documentation deployment occurs automatically when changes are merged to the default branch, using the `JamesIves/github-pages-deploy-action@v4` action with single-commit deployment to the `gh-pages` branch.

Sources: [.github/workflows/ci.yml(L50 - L56)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L50-L56)

## Architecture-Specific Considerations

When contributing architecture-specific code, ensure that:

1. **Conditional Compilation**: Use appropriate `cfg` attributes for target-specific code
2. **Cross-Architecture Compatibility**: Changes should not break compilation on other architectures
3. **Testing Coverage**: While unit tests only run on x86_64, ensure your code compiles cleanly on all targets
4. **Documentation**: Architecture-specific features should be clearly documented with appropriate `cfg_attr` annotations

The CI matrix validates all contributions across the full range of supported architectures, providing confidence that cross-platform compatibility is maintained.

Sources: [.github/workflows/ci.yml(L10 - L12)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L10-L12) [.github/workflows/ci.yml(L24 - L27)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/.github/workflows/ci.yml#L24-L27)