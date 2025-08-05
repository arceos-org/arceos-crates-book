# Build System and CI

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/.github/workflows/ci.yml)

This document covers the automated build system and continuous integration (CI) pipeline for the `kernel_guard` crate. The CI system is implemented using GitHub Actions and provides multi-architecture testing, code quality enforcement, and automated documentation deployment.

For information about setting up a local development environment, see [Development Environment](/arceos-org/kernel_guard/5.2-development-environment). For details about the specific architectures supported, see [Multi-Architecture Support](/arceos-org/kernel_guard/3-multi-architecture-support).

## CI Pipeline Overview

The kernel_guard project uses GitHub Actions for continuous integration, defined in a single workflow file that handles both build/test operations and documentation deployment. The pipeline is triggered on all push and pull request events.

**CI Pipeline Structure**

```mermaid
flowchart TD
TRIGGER["on: [push, pull_request]"]
JOBS["GitHub Actions Jobs"]
CI_JOB["ci job"]
DOC_JOB["doc job"]
MATRIX["strategy.matrix"]
CI_STEPS["CI Steps"]
DOC_PERMISSIONS["permissions: contents: write"]
DOC_STEPS["Documentation Steps"]
TOOLCHAIN["rust-toolchain: [nightly]"]
TARGETS["targets: [5 architectures]"]
CHECKOUT["actions/checkout@v4"]
TOOLCHAIN_SETUP["dtolnay/rust-toolchain@nightly"]
FORMAT_CHECK["cargo fmt --all -- --check"]
CLIPPY_CHECK["cargo clippy"]
BUILD_STEP["cargo build"]
TEST_STEP["cargo test"]
DOC_BUILD["cargo doc --no-deps --all-features"]
PAGES_DEPLOY["JamesIves/github-pages-deploy-action@v4"]

CI_JOB --> CI_STEPS
CI_JOB --> MATRIX
CI_STEPS --> BUILD_STEP
CI_STEPS --> CHECKOUT
CI_STEPS --> CLIPPY_CHECK
CI_STEPS --> FORMAT_CHECK
CI_STEPS --> TEST_STEP
CI_STEPS --> TOOLCHAIN_SETUP
DOC_JOB --> DOC_PERMISSIONS
DOC_JOB --> DOC_STEPS
DOC_STEPS --> DOC_BUILD
DOC_STEPS --> PAGES_DEPLOY
JOBS --> CI_JOB
JOBS --> DOC_JOB
MATRIX --> TARGETS
MATRIX --> TOOLCHAIN
TRIGGER --> JOBS
```

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/.github/workflows/ci.yml#L1-L56)

## CI Job Configuration

The primary `ci` job runs on `ubuntu-latest` and uses a matrix strategy to test multiple configurations simultaneously. The job is configured with `fail-fast: false` to ensure all matrix combinations are tested even if some fail.

### Matrix Strategy

The build matrix tests against multiple target architectures using the nightly Rust toolchain:

|Target Architecture|Purpose|
| --- | --- |
|x86_64-unknown-linux-gnu|Standard Linux testing and unit tests|
|x86_64-unknown-none|Bare metal x86_64|
|riscv64gc-unknown-none-elf|RISC-V 64-bit bare metal|
|aarch64-unknown-none-softfloat|ARM64 bare metal|
|loongarch64-unknown-none-softfloat|LoongArch64 bare metal|

**Matrix Configuration Flow**

```mermaid
flowchart TD
MATRIX_START["strategy.matrix"]
TOOLCHAIN_CONFIG["rust-toolchain: nightly"]
TARGET_CONFIG["targets: 5 architectures"]
NIGHTLY["nightly toolchain"]
X86_LINUX["x86_64-unknown-linux-gnu"]
X86_NONE["x86_64-unknown-none"]
RISCV["riscv64gc-unknown-none-elf"]
AARCH64["aarch64-unknown-none-softfloat"]
LOONG["loongarch64-unknown-none-softfloat"]
COMPONENTS["components: rust-src, clippy, rustfmt"]
TEST_ENABLED["Unit tests enabled"]
BUILD_ONLY["Build only"]

AARCH64 --> BUILD_ONLY
LOONG --> BUILD_ONLY
MATRIX_START --> TARGET_CONFIG
MATRIX_START --> TOOLCHAIN_CONFIG
NIGHTLY --> COMPONENTS
RISCV --> BUILD_ONLY
TARGET_CONFIG --> AARCH64
TARGET_CONFIG --> LOONG
TARGET_CONFIG --> RISCV
TARGET_CONFIG --> X86_LINUX
TARGET_CONFIG --> X86_NONE
TOOLCHAIN_CONFIG --> NIGHTLY
X86_LINUX --> TEST_ENABLED
X86_NONE --> BUILD_ONLY
```

Sources: [.github/workflows/ci.yml(L8 - L12)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/.github/workflows/ci.yml#L8-L12) [.github/workflows/ci.yml(L16 - L19)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/.github/workflows/ci.yml#L16-L19)

### Quality Assurance Steps

The CI job enforces code quality through a series of automated checks:

**Quality Assurance Workflow**

```mermaid
flowchart TD
SETUP["Rust Toolchain Setup"]
VERSION_CHECK["rustc --version --verbose"]
FORMAT_CHECK["cargo fmt --all -- --check"]
CLIPPY_CHECK["cargo clippy --target TARGET --all-features"]
BUILD_STEP["cargo build --target TARGET --all-features"]
TEST_DECISION["TARGET == x86_64-unknown-linux-gnu?"]
UNIT_TEST["cargo test --target TARGET -- --nocapture"]
COMPLETE["Job Complete"]
CLIPPY_ALLOW["-A clippy::new_without_default"]

BUILD_STEP --> TEST_DECISION
CLIPPY_CHECK --> BUILD_STEP
CLIPPY_CHECK --> CLIPPY_ALLOW
FORMAT_CHECK --> CLIPPY_CHECK
SETUP --> VERSION_CHECK
TEST_DECISION --> COMPLETE
TEST_DECISION --> UNIT_TEST
UNIT_TEST --> COMPLETE
VERSION_CHECK --> FORMAT_CHECK
```

The clippy step includes a specific allowance for the `new_without_default` lint, configured via the `-A clippy::new_without_default` flag.

Sources: [.github/workflows/ci.yml(L20 - L30)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/.github/workflows/ci.yml#L20-L30)

## Documentation Job and GitHub Pages

The `doc` job handles automated documentation generation and deployment to GitHub Pages. This job requires write permissions to the repository contents for deployment.

### Documentation Build Process

The documentation job uses specific environment variables and build flags to ensure high-quality documentation:

```yaml
RUSTDOCFLAGS: -D rustdoc::broken_intra_doc_links -D missing-docs
```

**Documentation Deployment Flow**

```mermaid
flowchart TD
DOC_TRIGGER["doc job trigger"]
PERMISSIONS_CHECK["permissions.contents: write"]
ENV_SETUP["RUSTDOCFLAGS environment"]
BRANCH_CHECK["default-branch variable"]
DOC_BUILD["cargo doc --no-deps --all-features"]
INDEX_GEN["Generate index.html redirect"]
BRANCH_CONDITION["github.ref == default-branch?"]
DEPLOY_PAGES["JamesIves/github-pages-deploy-action@v4"]
SKIP_DEPLOY["Skip deployment"]
PAGES_CONFIG["single-commit: truebranch: gh-pagesfolder: target/doc"]
STRICT_DOCS["Broken links fail buildMissing docs fail build"]

BRANCH_CHECK --> DOC_BUILD
BRANCH_CONDITION --> DEPLOY_PAGES
BRANCH_CONDITION --> SKIP_DEPLOY
DEPLOY_PAGES --> PAGES_CONFIG
DOC_BUILD --> INDEX_GEN
DOC_TRIGGER --> PERMISSIONS_CHECK
ENV_SETUP --> BRANCH_CHECK
ENV_SETUP --> STRICT_DOCS
INDEX_GEN --> BRANCH_CONDITION
PERMISSIONS_CHECK --> ENV_SETUP
```

The index.html generation uses a shell command to extract the crate name and create a redirect:

```
printf '<meta http-equiv="refresh" content="0;url=%s/index.html">' $(cargo tree | head -1 | cut -d' ' -f1) > target/doc/index.html
```

Sources: [.github/workflows/ci.yml(L32 - L56)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/.github/workflows/ci.yml#L32-L56) [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/.github/workflows/ci.yml#L40-L40) [.github/workflows/ci.yml(L46 - L48)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/.github/workflows/ci.yml#L46-L48)

## Build and Test Execution

The CI system executes different build and test strategies based on the target architecture:

### Target-Specific Behavior

```mermaid
flowchart TD
BUILD_START["cargo build --target TARGET --all-features"]
TARGET_CHECK["Check target architecture"]
LINUX_TARGET["x86_64-unknown-linux-gnu"]
BAREMETAL_X86["x86_64-unknown-none"]
BAREMETAL_RISCV["riscv64gc-unknown-none-elf"]
BAREMETAL_ARM["aarch64-unknown-none-softfloat"]
BAREMETAL_LOONG["loongarch64-unknown-none-softfloat"]
BUILD_SUCCESS["Build Success"]
TEST_CHECK["if: matrix.targets == 'x86_64-unknown-linux-gnu'"]
UNIT_TESTS["cargo test --target TARGET -- --nocapture"]
NO_TESTS["Skip unit tests"]
TEST_COMPLETE["All checks complete"]

BAREMETAL_ARM --> BUILD_SUCCESS
BAREMETAL_LOONG --> BUILD_SUCCESS
BAREMETAL_RISCV --> BUILD_SUCCESS
BAREMETAL_X86 --> BUILD_SUCCESS
BUILD_START --> TARGET_CHECK
BUILD_SUCCESS --> TEST_CHECK
LINUX_TARGET --> BUILD_SUCCESS
NO_TESTS --> TEST_COMPLETE
TARGET_CHECK --> BAREMETAL_ARM
TARGET_CHECK --> BAREMETAL_LOONG
TARGET_CHECK --> BAREMETAL_RISCV
TARGET_CHECK --> BAREMETAL_X86
TARGET_CHECK --> LINUX_TARGET
TEST_CHECK --> NO_TESTS
TEST_CHECK --> UNIT_TESTS
UNIT_TESTS --> TEST_COMPLETE
```

Unit tests are only executed on the `x86_64-unknown-linux-gnu` target because the bare metal targets cannot run standard Rust test frameworks.

Sources: [.github/workflows/ci.yml(L26 - L30)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/.github/workflows/ci.yml#L26-L30)

The CI pipeline ensures that the kernel_guard crate builds correctly across all supported architectures while maintaining code quality standards through automated formatting, linting, and testing.