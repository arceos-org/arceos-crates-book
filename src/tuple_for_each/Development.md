# Development

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.gitignore)
> * [tests/test_tuple_for_each.rs](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/tests/test_tuple_for_each.rs)

This page provides comprehensive guidance for contributors to the `tuple_for_each` crate, covering development workflow, testing strategies, and quality assurance processes. It focuses on the practical aspects of building, testing, and maintaining the codebase across multiple target architectures.

For detailed information about testing patterns and integration test structure, see [Testing](/arceos-org/tuple_for_each/4.1-testing). For CI/CD pipeline configuration and deployment processes, see [CI/CD Pipeline](/arceos-org/tuple_for_each/4.2-cicd-pipeline).

## Development Environment Setup

The `tuple_for_each` crate requires a Rust nightly toolchain with specific components for cross-platform development and quality checks.

### Required Toolchain Components

|Component|Purpose|
| --- | --- |
|rust-src|Source code for cross-compilation|
|clippy|Linting and static analysis|
|rustfmt|Code formatting|

### Supported Target Architectures

The project supports multiple target architectures to ensure compatibility across embedded and systems programming environments:

```mermaid
flowchart TD
subgraph subGraph2["Testing Strategy"]
    FULL_TEST["Full test suite"]
    BUILD_ONLY["Build verification only"]
end
subgraph subGraph1["Target Architectures"]
    X86_LINUX["x86_64-unknown-linux-gnu"]
    X86_NONE["x86_64-unknown-none"]
    RISCV["riscv64gc-unknown-none-elf"]
    ARM["aarch64-unknown-none-softfloat"]
end
subgraph Toolchain["Toolchain"]
    NIGHTLY["nightly toolchain"]
end

ARM --> BUILD_ONLY
NIGHTLY --> ARM
NIGHTLY --> RISCV
NIGHTLY --> X86_LINUX
NIGHTLY --> X86_NONE
RISCV --> BUILD_ONLY
X86_LINUX --> FULL_TEST
X86_NONE --> BUILD_ONLY
```

**Target Architecture Support Matrix**

Sources: [.github/workflows/ci.yml(L12)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L12-L12) [.github/workflows/ci.yml(L29 - L30)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L29-L30)

## Development Workflow

### Quality Gates Pipeline

The development process enforces multiple quality gates before code integration:

```mermaid
flowchart TD
subgraph subGraph0["Parallel Execution"]
    TARGET1["x86_64-unknown-linux-gnu"]
    TARGET2["x86_64-unknown-none"]
    TARGET3["riscv64gc-unknown-none-elf"]
    TARGET4["aarch64-unknown-none-softfloat"]
end
COMMIT["Code Commit"]
FMT_CHECK["cargo fmt --all -- --check"]
CLIPPY_CHECK["cargo clippy --target TARGET --all-features"]
BUILD["cargo build --target TARGET --all-features"]
UNIT_TEST["cargo test --target x86_64-unknown-linux-gnu"]
SUCCESS["Integration Complete"]

BUILD --> TARGET1
BUILD --> TARGET2
BUILD --> TARGET3
BUILD --> TARGET4
BUILD --> UNIT_TEST
CLIPPY_CHECK --> BUILD
COMMIT --> FMT_CHECK
FMT_CHECK --> CLIPPY_CHECK
TARGET1 --> SUCCESS
TARGET2 --> SUCCESS
TARGET3 --> SUCCESS
TARGET4 --> SUCCESS
UNIT_TEST --> SUCCESS
```

**Development Quality Pipeline**

Sources: [.github/workflows/ci.yml(L22 - L30)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L22-L30)

### Local Development Commands

|Command|Purpose|
| --- | --- |
|cargo fmt --all -- --check|Verify code formatting|
|cargo clippy --all-features|Run linting checks|
|cargo build --all-features|Build for host target|
|cargo test -- --nocapture|Run integration tests|

## Testing Architecture

The testing strategy focuses on integration tests that verify the generated macro functionality across different usage patterns.

### Test Structure Overview

```mermaid
flowchart TD
subgraph subGraph3["Test Module Structure"]
    TEST_FILE["tests/test_tuple_for_each.rs"]
    subgraph subGraph2["Test Functions"]
        TEST_FOR_EACH["test_for_each()"]
        TEST_FOR_EACH_MUT["test_for_each_mut()"]
        TEST_ENUMERATE["test_enumerate()"]
        TEST_ENUMERATE_MUT["test_enumerate_mut()"]
    end
    subgraph subGraph1["Test Targets"]
        PAIR_STRUCT["Pair(A, B)"]
        TUPLE_STRUCT["Tuple(A, B, C)"]
    end
    subgraph subGraph0["Test Infrastructure"]
        BASE_TRAIT["Base trait"]
        IMPL_A["A impl Base"]
        IMPL_B["B impl Base"]
        IMPL_C["C impl Base"]
    end
end

BASE_TRAIT --> IMPL_A
BASE_TRAIT --> IMPL_B
BASE_TRAIT --> IMPL_C
IMPL_A --> PAIR_STRUCT
IMPL_B --> PAIR_STRUCT
IMPL_B --> TUPLE_STRUCT
IMPL_C --> TUPLE_STRUCT
PAIR_STRUCT --> TEST_ENUMERATE_MUT
PAIR_STRUCT --> TEST_FOR_EACH
TEST_FILE --> BASE_TRAIT
TUPLE_STRUCT --> TEST_ENUMERATE
TUPLE_STRUCT --> TEST_FOR_EACH_MUT
```

**Integration Test Architecture**

### Test Function Coverage

|Test Function|Target Struct|Generated Macro|Validation|
| --- | --- | --- | --- |
|test_for_each|Pair|pair_for_each!|Iteration count, method calls|
|test_for_each_mut|Tuple|tuple_for_each!|Mutable iteration|
|test_enumerate|Tuple|tuple_enumerate!|Index validation|
|test_enumerate_mut|Pair|pair_enumerate!|Mutable enumeration|

Sources: [tests/test_tuple_for_each.rs(L50 - L106)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/tests/test_tuple_for_each.rs#L50-L106)

## CI/CD Infrastructure

### Workflow Jobs Architecture

```mermaid
flowchart TD
subgraph subGraph3["Documentation Job"]
    DOC_JOB["doc job"]
    BUILD_DOCS["cargo doc --no-deps --all-features"]
    DEPLOY_PAGES["Deploy to GitHub Pages"]
end
subgraph subGraph2["CI Job Matrix"]
    CI_JOB["ci job"]
    subgraph subGraph1["Matrix Strategy"]
        RUST_NIGHTLY["rust-toolchain: nightly"]
        TARGETS_MATRIX["targets: [4 architectures]"]
    end
end
subgraph subGraph0["GitHub Actions Triggers"]
    PUSH["push event"]
    PR["pull_request event"]
end

BUILD_DOCS --> DEPLOY_PAGES
CI_JOB --> RUST_NIGHTLY
CI_JOB --> TARGETS_MATRIX
DOC_JOB --> BUILD_DOCS
PR --> CI_JOB
PUSH --> CI_JOB
PUSH --> DOC_JOB
```

**CI/CD Job Architecture**

### Documentation Deployment Process

The documentation pipeline includes automated deployment to GitHub Pages with strict quality requirements:

|Environment Variable|Purpose|
| --- | --- |
|RUSTDOCFLAGS: -D rustdoc::broken_intra_doc_links -D missing-docs|Enforce complete documentation|
|default-branch|Control deployment target|

Sources: [.github/workflows/ci.yml(L32 - L55)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L32-L55) [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L40-L40)

## Multi-Target Build Strategy

### Cross-Compilation Support

The crate supports embedded and systems programming environments through comprehensive cross-compilation testing:

```mermaid
flowchart TD
subgraph subGraph2["Build Verification"]
    FORMAT_CHECK["Format verification"]
    LINT_CHECK["Clippy linting"]
    BUILD_CHECK["Compilation check"]
    UNIT_TESTING["Unit test execution"]
end
subgraph subGraph1["Target Platforms"]
    LINUX_GNU["x86_64-unknown-linux-gnuStandard Linux"]
    BARE_METAL_X86["x86_64-unknown-noneBare metal x86"]
    RISCV_EMBEDDED["riscv64gc-unknown-none-elfRISC-V embedded"]
    ARM_EMBEDDED["aarch64-unknown-none-softfloatARM embedded"]
end
subgraph subGraph0["Host Environment"]
    UBUNTU["ubuntu-latest runner"]
    NIGHTLY_TOOLCHAIN["Rust nightly toolchain"]
end

ARM_EMBEDDED --> BUILD_CHECK
ARM_EMBEDDED --> FORMAT_CHECK
ARM_EMBEDDED --> LINT_CHECK
BARE_METAL_X86 --> BUILD_CHECK
BARE_METAL_X86 --> FORMAT_CHECK
BARE_METAL_X86 --> LINT_CHECK
LINUX_GNU --> BUILD_CHECK
LINUX_GNU --> FORMAT_CHECK
LINUX_GNU --> LINT_CHECK
LINUX_GNU --> UNIT_TESTING
NIGHTLY_TOOLCHAIN --> ARM_EMBEDDED
NIGHTLY_TOOLCHAIN --> BARE_METAL_X86
NIGHTLY_TOOLCHAIN --> LINUX_GNU
NIGHTLY_TOOLCHAIN --> RISCV_EMBEDDED
RISCV_EMBEDDED --> BUILD_CHECK
RISCV_EMBEDDED --> FORMAT_CHECK
RISCV_EMBEDDED --> LINT_CHECK
UBUNTU --> NIGHTLY_TOOLCHAIN
```

**Multi-Target Build Matrix**

### Testing Limitations by Target

Only the `x86_64-unknown-linux-gnu` target supports full test execution due to standard library dependencies in the test environment. Embedded targets (`*-none-*`) undergo build verification to ensure compilation compatibility without runtime testing.

Sources: [.github/workflows/ci.yml(L8 - L30)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L8-L30)