# CI/CD Pipeline

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/handler_table/blob/036a12c4/.github/workflows/ci.yml)

This page documents the Continuous Integration and Continuous Deployment (CI/CD) pipeline for the `handler_table` crate. The pipeline ensures code quality, compatibility across multiple architectures, and automated documentation deployment through GitHub Actions workflows.

For information about building and testing locally during development, see [Building and Testing](/arceos-org/handler_table/4.1-building-and-testing).

## Pipeline Overview

The CI/CD system consists of two primary GitHub Actions workflows that execute on push and pull request events. The pipeline validates code across multiple target architectures and automatically deploys documentation to GitHub Pages.

### CI/CD Workflow Architecture

```mermaid
flowchart TD
subgraph subGraph3["Documentation Pipeline"]
    DocWorkflow["doc job"]
    DocCheckout["actions/checkout@v4"]
    DocBuild["cargo doc --no-deps"]
    IndexGen["generate index.html redirect"]
    Deploy["JamesIves/github-pages-deploy-action@v4"]
    Pages["GitHub Pages (gh-pages branch)"]
end
subgraph subGraph2["CI Steps"]
    Checkout["actions/checkout@v4"]
    Toolchain["dtolnay/rust-toolchain@nightly"]
    Format["cargo fmt --check"]
    Clippy["cargo clippy"]
    Build["cargo build"]
    Test["cargo test (linux only)"]
end
subgraph subGraph1["Target Architectures"]
    LinuxGNU["x86_64-unknown-linux-gnu"]
    BareX86["x86_64-unknown-none"]
    RISCV["riscv64gc-unknown-none-elf"]
    ARM["aarch64-unknown-none-softfloat"]
end
subgraph subGraph0["CI Job Matrix"]
    CIWorkflow["ci job"]
    Matrix["strategy.matrix"]
    RustNightly["rust-toolchain: nightly"]
    Targets["4 target architectures"]
end
Triggers["push/pull_request events"]

Build --> Test
CIWorkflow --> Matrix
Checkout --> Toolchain
Clippy --> Build
Deploy --> Pages
DocBuild --> IndexGen
DocCheckout --> DocBuild
DocWorkflow --> DocCheckout
Format --> Clippy
IndexGen --> Deploy
Matrix --> RustNightly
Matrix --> Targets
RustNightly --> Checkout
Targets --> ARM
Targets --> BareX86
Targets --> LinuxGNU
Targets --> RISCV
Toolchain --> Format
Triggers --> CIWorkflow
Triggers --> DocWorkflow
```

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/.github/workflows/ci.yml#L1-L56)

## Multi-Architecture Testing Strategy

The CI pipeline validates compatibility across four distinct target architectures to ensure the `handler_table` crate functions correctly in diverse embedded and system environments.

### Architecture Testing Matrix

|Target|Environment|Test Coverage|
| --- | --- | --- |
|x86_64-unknown-linux-gnu|Standard Linux|Full (fmt, clippy, build, test)|
|x86_64-unknown-none|Bare metal x86_64|Build and lint only|
|riscv64gc-unknown-none-elf|RISC-V bare metal|Build and lint only|
|aarch64-unknown-none-softfloat|ARM64 bare metal|Build and lint only|

The pipeline uses a fail-fast strategy set to `false`, ensuring all target combinations are tested even if one fails.

### Target-Specific Execution Flow

```mermaid
flowchart TD
subgraph subGraph2["Toolchain Components"]
    RustSrc["rust-src"]
    ClippyComp["clippy"]
    RustfmtComp["rustfmt"]
    TargetSpec["target-specific toolchain"]
end
subgraph subGraph1["Per-Target Steps"]
    Setup["Setup: checkout + toolchain + components"]
    VersionCheck["rustc --version --verbose"]
    FormatCheck["cargo fmt --all -- --check"]
    ClippyLint["cargo clippy --target TARGET --all-features"]
    BuildStep["cargo build --target TARGET --all-features"]
    TestStep["cargo test (if x86_64-unknown-linux-gnu)"]
end
subgraph subGraph0["Matrix Strategy"]
    FailFast["fail-fast: false"]
    NightlyToolchain["rust-toolchain: [nightly]"]
    FourTargets["targets: [4 architectures]"]
end

BuildStep --> TestStep
ClippyLint --> BuildStep
FailFast --> Setup
FormatCheck --> ClippyLint
FourTargets --> Setup
NightlyToolchain --> Setup
Setup --> ClippyComp
Setup --> RustSrc
Setup --> RustfmtComp
Setup --> TargetSpec
Setup --> VersionCheck
VersionCheck --> FormatCheck
```

Sources: [.github/workflows/ci.yml(L8 - L12)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/.github/workflows/ci.yml#L8-L12) [.github/workflows/ci.yml(L25 - L30)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/.github/workflows/ci.yml#L25-L30)

## Documentation Deployment

The documentation pipeline builds API documentation using `cargo doc` and deploys it to GitHub Pages. The deployment only occurs on pushes to the default branch, while documentation building is attempted on all branches and pull requests.

### Documentation Build Process

The documentation job includes specific configurations for documentation quality:

* **Environment Variables**: `RUSTDOCFLAGS` set to `-D rustdoc::broken_intra_doc_links -D missing-docs` to enforce documentation standards
* **Build Command**: `cargo doc --no-deps --all-features` to generate comprehensive API documentation
* **Index Generation**: Automatic creation of a redirect `index.html` pointing to the crate's documentation root

### Documentation Deployment Flow

```mermaid
flowchart TD
subgraph subGraph2["Error Handling"]
    ContinueOnError["continue-on-error for non-default branches"]
end
subgraph subGraph1["Conditional Deployment"]
    BranchCheck["github.ref == default-branch?"]
    DeployAction["JamesIves/github-pages-deploy-action@v4"]
    SkipDeploy["Skip deployment"]
    SingleCommit["single-commit: true"]
    GHPages["branch: gh-pages"]
    TargetDoc["folder: target/doc"]
    PagesDeployment["GitHub Pages Site"]
end
subgraph subGraph0["Documentation Build"]
    DocJob["doc job"]
    DocSetup["checkout + rust nightly"]
    DocFlags["RUSTDOCFLAGS environment"]
    CargoDocs["cargo doc --no-deps --all-features"]
    IndexHTML["generate target/doc/index.html redirect"]
end
DocTrigger["Push/PR to any branch"]

BranchCheck --> DeployAction
BranchCheck --> SkipDeploy
CargoDocs --> ContinueOnError
CargoDocs --> IndexHTML
DeployAction --> GHPages
DeployAction --> SingleCommit
DeployAction --> TargetDoc
DocFlags --> CargoDocs
DocJob --> DocSetup
DocSetup --> DocFlags
DocTrigger --> DocJob
GHPages --> PagesDeployment
IndexHTML --> BranchCheck
SingleCommit --> PagesDeployment
TargetDoc --> PagesDeployment
```

Sources: [.github/workflows/ci.yml(L32 - L56)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/.github/workflows/ci.yml#L32-L56) [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/.github/workflows/ci.yml#L40-L40) [.github/workflows/ci.yml(L47 - L48)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/.github/workflows/ci.yml#L47-L48)

## Workflow Job Structure

The CI/CD pipeline is organized into two distinct GitHub Actions jobs with different responsibilities and execution contexts.

### Job Configuration Details

**CI Job (`ci`)**:

* **Runner**: `ubuntu-latest`
* **Strategy**: Matrix execution across rust toolchain and target combinations
* **Purpose**: Code quality validation and multi-architecture compatibility testing

**Documentation Job (`doc`)**:

* **Runner**: `ubuntu-latest`
* **Strategy**: Single execution context
* **Permissions**: `contents: write` for GitHub Pages deployment
* **Purpose**: API documentation generation and deployment

### Complete Workflow Structure

```mermaid
flowchart TD
subgraph subGraph4["Doc Job Steps"]
    DocStep1["actions/checkout@v4"]
    DocStep2["dtolnay/rust-toolchain@nightly"]
    DocStep3["cargo doc --no-deps --all-features"]
    DocStep4["generate index.html redirect"]
    DocStep5["JamesIves/github-pages-deploy-action@v4"]
end
subgraph subGraph3["Doc Job Configuration"]
    DocRunner["runs-on: ubuntu-latest"]
    DocPerms["permissions: contents: write"]
    DocEnv["RUSTDOCFLAGS environment"]
end
subgraph subGraph2["CI Job Steps"]
    CIStep1["actions/checkout@v4"]
    CIStep2["dtolnay/rust-toolchain@nightly"]
    CIStep3["rustc --version --verbose"]
    CIStep4["cargo fmt --all -- --check"]
    CIStep5["cargo clippy --target TARGET"]
    CIStep6["cargo build --target TARGET"]
    CIStep7["cargo test (conditional)"]
end
subgraph subGraph1["CI Job Configuration"]
    CIRunner["runs-on: ubuntu-latest"]
    CIStrategy["strategy.fail-fast: false"]
    CIMatrix["matrix: nightly × 4 targets"]
end
subgraph subGraph0["Workflow Triggers"]
    PushEvent["on: push"]
    PREvent["on: pull_request"]
end

CIMatrix --> CIStep1
CIRunner --> CIStrategy
CIStep1 --> CIStep2
CIStep2 --> CIStep3
CIStep3 --> CIStep4
CIStep4 --> CIStep5
CIStep5 --> CIStep6
CIStep6 --> CIStep7
CIStrategy --> CIMatrix
DocEnv --> DocStep1
DocPerms --> DocEnv
DocRunner --> DocPerms
DocStep1 --> DocStep2
DocStep2 --> DocStep3
DocStep3 --> DocStep4
DocStep4 --> DocStep5
PREvent --> CIRunner
PREvent --> DocRunner
PushEvent --> CIRunner
PushEvent --> DocRunner
```

Sources: [.github/workflows/ci.yml(L1 - L31)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/.github/workflows/ci.yml#L1-L31) [.github/workflows/ci.yml(L32 - L56)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/.github/workflows/ci.yml#L32-L56) [.github/workflows/ci.yml(L36 - L37)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/.github/workflows/ci.yml#L36-L37)