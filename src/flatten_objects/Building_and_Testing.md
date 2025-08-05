# Building and Testing

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/flatten_objects/blob/ac0a74b9/.github/workflows/ci.yml)

This document details the continuous integration pipeline, multi-target build system, testing strategy, and development workflow for the `flatten_objects` crate. It covers the automated quality assurance processes, supported target platforms, and local development practices.

For information about project configuration and development environment setup, see [Project Configuration](/arceos-org/flatten_objects/5.2-project-configuration).

## CI Pipeline Overview

The `flatten_objects` crate uses GitHub Actions for continuous integration with two primary workflows: code quality validation and documentation deployment. The pipeline ensures compatibility across multiple target platforms while maintaining code quality standards.

### CI Workflow Architecture

```mermaid
flowchart TD
subgraph subGraph3["Documentation Job"]
    doc_build["cargo doc --no-deps --all-features"]
    pages_deploy["JamesIves/github-pages-deploy-action@v4"]
end
subgraph subGraph2["Build Steps"]
    checkout["actions/checkout@v4"]
    toolchain["dtolnay/rust-toolchain@nightly"]
    rustc_version["rustc --version"]
    cargo_fmt["cargo fmt --all --check"]
    cargo_clippy["cargo clippy --target TARGET"]
    cargo_build["cargo build --target TARGET"]
    cargo_test["cargo test --target x86_64-unknown-linux-gnu"]
end
subgraph subGraph1["CI Job Matrix"]
    strategy["strategy.matrix"]
    rust_nightly["rust-toolchain: nightly"]
    target_linux["x86_64-unknown-linux-gnu"]
    target_none["x86_64-unknown-none"]
    target_riscv["riscv64gc-unknown-none-elf"]
    target_arm["aarch64-unknown-none-softfloat"]
end
subgraph subGraph0["GitHub Events"]
    push["push"]
    pr["pull_request"]
end

cargo_build --> cargo_test
cargo_clippy --> cargo_build
cargo_fmt --> cargo_clippy
checkout --> toolchain
doc_build --> pages_deploy
pr --> strategy
push --> doc_build
push --> strategy
rust_nightly --> target_arm
rust_nightly --> target_linux
rust_nightly --> target_none
rust_nightly --> target_riscv
rustc_version --> cargo_fmt
strategy --> rust_nightly
target_arm --> checkout
target_linux --> checkout
target_none --> checkout
target_riscv --> checkout
toolchain --> rustc_version
```

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/flatten_objects/blob/ac0a74b9/.github/workflows/ci.yml#L1-L56)

### Job Matrix Configuration

The CI system uses a matrix strategy to test against multiple target platforms simultaneously:

|Target Platform|Purpose|Test Execution|
| --- | --- | --- |
|x86_64-unknown-linux-gnu|Standard Linux development|Full test suite|
|x86_64-unknown-none|Bare metal x86_64|Build only|
|riscv64gc-unknown-none-elf|RISC-V bare metal|Build only|
|aarch64-unknown-none-softfloat|ARM64 bare metal|Build only|

Sources: [.github/workflows/ci.yml(L12)&emsp;](https://github.com/arceos-org/flatten_objects/blob/ac0a74b9/.github/workflows/ci.yml#L12-L12)

## Multi-Target Build System

The crate supports multiple target architectures to ensure compatibility with various embedded and bare metal environments. The build system validates compilation across all supported targets.

### Target Platform Validation Flow

```mermaid
flowchart TD
subgraph subGraph3["Test Execution"]
    test_linux["cargo test --target x86_64-unknown-linux-gnu"]
    test_condition["if matrix.targets == 'x86_64-unknown-linux-gnu'"]
end
subgraph subGraph2["Build Validation"]
    build_linux["cargo build --target x86_64-unknown-linux-gnu"]
    build_none["cargo build --target x86_64-unknown-none"]
    build_riscv["cargo build --target riscv64gc-unknown-none-elf"]
    build_arm["cargo build --target aarch64-unknown-none-softfloat"]
end
subgraph subGraph1["Quality Checks"]
    fmt_check["cargo fmt --all --check"]
    clippy_check["cargo clippy --target TARGET"]
end
subgraph subGraph0["Rust Toolchain Setup"]
    nightly["nightly toolchain"]
    components["rust-src, clippy, rustfmt"]
    targets["target installation"]
end

build_linux --> test_condition
clippy_check --> build_arm
clippy_check --> build_linux
clippy_check --> build_none
clippy_check --> build_riscv
components --> targets
fmt_check --> clippy_check
nightly --> components
targets --> fmt_check
test_condition --> test_linux
```

Sources: [.github/workflows/ci.yml(L15 - L30)&emsp;](https://github.com/arceos-org/flatten_objects/blob/ac0a74b9/.github/workflows/ci.yml#L15-L30)

### Target-Specific Build Commands

Each target platform uses identical build commands but with different target specifications:

* **Format Check**: `cargo fmt --all -- --check` validates code formatting consistency
* **Linting**: `cargo clippy --target ${{ matrix.targets }} --all-features -- -A clippy::new_without_default`
* **Build**: `cargo build --target ${{ matrix.targets }} --all-features`
* **Testing**: `cargo test --target ${{ matrix.targets }} -- --nocapture` (Linux only)

Sources: [.github/workflows/ci.yml(L23 - L30)&emsp;](https://github.com/arceos-org/flatten_objects/blob/ac0a74b9/.github/workflows/ci.yml#L23-L30)

## Testing Strategy

The testing approach focuses on functional validation while accommodating the constraints of bare metal target platforms.

### Test Execution Matrix

```mermaid
flowchart TD
subgraph subGraph2["Execution Strategy"]
    linux_only["x86_64-unknown-linux-gnu only"]
    build_only["Build validation only"]
    nocapture["--nocapture flag"]
end
subgraph subGraph1["Test Types"]
    unit_tests["Unit Tests"]
    integration_tests["Integration Tests"]
    doc_tests["Documentation Tests"]
end
subgraph subGraph0["Test Environments"]
    hosted["Hosted Environment"]
    bare_metal["Bare Metal Targets"]
end

bare_metal --> build_only
doc_tests --> linux_only
hosted --> doc_tests
hosted --> integration_tests
hosted --> unit_tests
integration_tests --> linux_only
linux_only --> nocapture
unit_tests --> linux_only
```

Sources: [.github/workflows/ci.yml(L28 - L30)&emsp;](https://github.com/arceos-org/flatten_objects/blob/ac0a74b9/.github/workflows/ci.yml#L28-L30)

### Test Execution Conditions

Tests are executed only on the `x86_64-unknown-linux-gnu` target due to the following constraints:

* Bare metal targets lack test harness support
* `no_std` environments have limited testing infrastructure
* The `--nocapture` flag provides detailed test output for debugging

The conditional test execution is implemented using GitHub Actions matrix conditions:

```css
if: ${{ matrix.targets == 'x86_64-unknown-linux-gnu' }}
```

Sources: [.github/workflows/ci.yml(L29)&emsp;](https://github.com/arceos-org/flatten_objects/blob/ac0a74b9/.github/workflows/ci.yml#L29-L29)

## Code Quality Checks

The CI pipeline enforces code quality through automated formatting and linting checks that run before build validation.

### Quality Assurance Pipeline

```mermaid
flowchart TD
subgraph subGraph2["Quality Gates"]
    fmt_gate["Format validation"]
    lint_gate["Lint validation"]
    allowlist["clippy::new_without_default"]
end
subgraph subGraph1["Quality Commands"]
    fmt_cmd["cargo fmt --all --check"]
    clippy_cmd["cargo clippy --all-features"]
    version_cmd["rustc --version --verbose"]
end
subgraph subGraph0["Code Quality Tools"]
    rustfmt["rustfmt"]
    clippy["clippy"]
    rustc["rustc"]
end

clippy --> clippy_cmd
clippy_cmd --> lint_gate
fmt_cmd --> fmt_gate
lint_gate --> allowlist
rustc --> version_cmd
rustfmt --> fmt_cmd
```

Sources: [.github/workflows/ci.yml(L20 - L25)&emsp;](https://github.com/arceos-org/flatten_objects/blob/ac0a74b9/.github/workflows/ci.yml#L20-L25)

### Clippy Configuration

The linting process uses specific allowlist configurations:

* **Suppressed Warning**: `clippy::new_without_default` is allowed since the crate provides specialized constructors
* **Target-Specific**: Clippy runs against each target platform independently
* **Feature Complete**: `--all-features` ensures comprehensive linting coverage

Sources: [.github/workflows/ci.yml(L25)&emsp;](https://github.com/arceos-org/flatten_objects/blob/ac0a74b9/.github/workflows/ci.yml#L25-L25)

## Documentation Build and Deployment

The documentation system automatically builds and deploys API documentation to GitHub Pages for the main branch.

### Documentation Workflow

```mermaid
flowchart TD
subgraph subGraph3["Error Handling"]
    continue_on_error["continue-on-error"]
    branch_condition["github.ref == env.default-branch"]
end
subgraph Deployment["Deployment"]
    pages_action["JamesIves/github-pages-deploy-action@v4"]
    gh_pages["gh-pages branch"]
    single_commit["single-commit: true"]
end
subgraph subGraph1["Build Process"]
    cargo_doc["cargo doc --no-deps --all-features"]
    doc_tree["cargo tree"]
    index_html["index.html redirect"]
end
subgraph subGraph0["Documentation Job"]
    doc_trigger["default-branch push"]
    doc_permissions["contents: write"]
    rustdoc_flags["RUSTDOCFLAGS environment"]
end

branch_condition --> pages_action
cargo_doc --> continue_on_error
cargo_doc --> doc_tree
doc_permissions --> rustdoc_flags
doc_tree --> index_html
doc_trigger --> doc_permissions
index_html --> branch_condition
pages_action --> gh_pages
pages_action --> single_commit
rustdoc_flags --> cargo_doc
```

Sources: [.github/workflows/ci.yml(L32 - L56)&emsp;](https://github.com/arceos-org/flatten_objects/blob/ac0a74b9/.github/workflows/ci.yml#L32-L56)

### Documentation Configuration

The documentation build process includes several important configurations:

* **Error Detection**: `RUSTDOCFLAGS: -D rustdoc::broken_intra_doc_links -D missing-docs`
* **Dependency Exclusion**: `--no-deps` focuses on crate-specific documentation
* **Branch Protection**: Deployment only occurs on the default branch
* **Index Generation**: Automatic redirect to main crate documentation

Sources: [.github/workflows/ci.yml(L40 - L48)&emsp;](https://github.com/arceos-org/flatten_objects/blob/ac0a74b9/.github/workflows/ci.yml#L40-L48)

## Local Development Workflow

For local development, developers should follow the same quality checks used in CI to ensure compatibility before pushing changes.

### Local Development Commands

|Command|Purpose|Target Requirement|
| --- | --- | --- |
|cargo fmt --all --check|Validate formatting|Any|
|cargo clippy --all-features|Run linting|Any|
|cargo build --target <TARGET>|Build validation|Specific target|
|cargo test|Execute tests|Linux host only|
|cargo doc --no-deps|Generate documentation|Any|

### Recommended Development Sequence

```mermaid
flowchart TD
subgraph subGraph2["Commit Process"]
    git_commit["git commit"]
    ci_trigger["CI pipeline trigger"]
end
subgraph subGraph1["Pre-commit Validation"]
    fmt_check_local["cargo fmt --all --check"]
    clippy_check_local["cargo clippy --all-features"]
    multi_target["Multi-target builds"]
end
subgraph subGraph0["Local Development"]
    code_change["Code changes"]
    fmt_local["cargo fmt --all"]
    clippy_local["cargo clippy --fix"]
    build_local["cargo build"]
    test_local["cargo test"]
end

build_local --> test_local
clippy_check_local --> multi_target
clippy_local --> build_local
code_change --> fmt_local
fmt_check_local --> clippy_check_local
fmt_local --> clippy_local
git_commit --> ci_trigger
multi_target --> git_commit
test_local --> fmt_check_local
```

Sources: [.github/workflows/ci.yml(L22 - L27)&emsp;](https://github.com/arceos-org/flatten_objects/blob/ac0a74b9/.github/workflows/ci.yml#L22-L27)