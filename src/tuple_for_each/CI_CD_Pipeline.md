# CI/CD Pipeline

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml)

This document covers the continuous integration and continuous deployment (CI/CD) infrastructure for the `tuple_for_each` crate. The pipeline is implemented using GitHub Actions and provides automated quality assurance, multi-target compilation, and documentation deployment.

The CI/CD system ensures code quality across multiple target architectures and automatically publishes documentation. For information about local development testing, see [Testing](/arceos-org/tuple_for_each/4.1-testing).

## Pipeline Overview

The CI/CD pipeline consists of two primary jobs defined in the GitHub Actions workflow: the `ci` job for quality assurance and multi-target builds, and the `doc` job for documentation generation and deployment.

### CI/CD Architecture

```mermaid
flowchart TD
Trigger["GitHub Event Triggerpush | pull_request"]
Pipeline["GitHub Actions Workflow.github/workflows/ci.yml"]
CIJob["ci JobQuality Assurance"]
DocJob["doc JobDocumentation"]
Matrix["Matrix Strategyrust-toolchain: nightlytargets: 4 platforms"]
Target1["x86_64-unknown-linux-gnuFull Testing"]
Target2["x86_64-unknown-noneBuild Only"]
Target3["riscv64gc-unknown-none-elfBuild Only"]
Target4["aarch64-unknown-none-softfloatBuild Only"]
DocBuild["cargo doc --no-deps --all-features"]
Deploy["GitHub Pages Deploymentgh-pages branch"]
QualityGates["Quality GatesFormat | Clippy | Build | Test"]
BuildOnly["Build GateFormat | Clippy | Build"]

CIJob --> Matrix
DocJob --> Deploy
DocJob --> DocBuild
Matrix --> Target1
Matrix --> Target2
Matrix --> Target3
Matrix --> Target4
Pipeline --> CIJob
Pipeline --> DocJob
Target1 --> QualityGates
Target2 --> BuildOnly
Target3 --> BuildOnly
Target4 --> BuildOnly
Trigger --> Pipeline
```

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L1-L56)

## CI Job Workflow

The `ci` job implements a comprehensive quality assurance pipeline that runs across multiple target architectures using a matrix strategy.

### Matrix Build Strategy

The CI job uses a fail-fast strategy disabled to ensure all target builds are attempted even if one fails:

|Configuration|Value|
| --- | --- |
|Runner|ubuntu-latest|
|Rust Toolchain|nightly|
|Target Architectures|4 platforms|
|Fail Fast|false|

The target matrix includes both hosted and embedded platforms:

```mermaid
flowchart TD
Nightly["rust-toolchain: nightlywith components:rust-src, clippy, rustfmt"]
Targets["Target Matrix"]
Linux["x86_64-unknown-linux-gnuStandard Linux TargetFull Testing Enabled"]
BareMetal["x86_64-unknown-noneBare Metal x86_64Build Only"]
RISCV["riscv64gc-unknown-none-elfRISC-V 64-bitBuild Only"]
ARM["aarch64-unknown-none-softfloatARM64 Bare MetalBuild Only"]
FullPipeline["Format Check → Clippy → Build → Test"]
BuildPipeline["Format Check → Clippy → Build"]

ARM --> BuildPipeline
BareMetal --> BuildPipeline
Linux --> FullPipeline
Nightly --> Targets
RISCV --> BuildPipeline
Targets --> ARM
Targets --> BareMetal
Targets --> Linux
Targets --> RISCV
```

Sources: [.github/workflows/ci.yml(L8 - L12)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L8-L12) [.github/workflows/ci.yml(L15 - L19)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L15-L19)

### Quality Gates

The CI pipeline implements several sequential quality gates:

#### 1. Code Format Verification

```
cargo fmt --all -- --check
```

Ensures all code follows consistent formatting standards using `rustfmt`.

#### 2. Linting with Clippy

```css
cargo clippy --target ${{ matrix.targets }} --all-features -- -A clippy::new_without_default
```

Performs static analysis with the `new_without_default` warning specifically allowed.

#### 3. Compilation

```
cargo build --target ${{ matrix.targets }} --all-features
```

Validates successful compilation for each target architecture.

#### 4. Unit Testing

```
cargo test --target ${{ matrix.targets }} -- --nocapture
```

Executes the test suite, but only for the `x86_64-unknown-linux-gnu` target due to testing infrastructure limitations on bare-metal targets.

### CI Job Steps Flow

```mermaid
sequenceDiagram
    participant GitHubActions as "GitHub Actions"
    participant ubuntulatestRunner as "ubuntu-latest Runner"
    participant RustToolchain as "Rust Toolchain"
    participant SourceCode as "Source Code"

    GitHubActions ->> ubuntulatestRunner: "actions/checkout@v4"
    ubuntulatestRunner ->> SourceCode: Clone repository
    GitHubActions ->> ubuntulatestRunner: "dtolnay/rust-toolchain@nightly"
    ubuntulatestRunner ->> RustToolchain: Install nightly + components + targets
    ubuntulatestRunner ->> RustToolchain: "rustc --version --verbose"
    RustToolchain -->> ubuntulatestRunner: Version info
    ubuntulatestRunner ->> SourceCode: "cargo fmt --all -- --check"
    SourceCode -->> ubuntulatestRunner: Format validation result
    ubuntulatestRunner ->> SourceCode: "cargo clippy --target TARGET"
    SourceCode -->> ubuntulatestRunner: Lint analysis result
    ubuntulatestRunner ->> SourceCode: "cargo build --target TARGET"
    SourceCode -->> ubuntulatestRunner: Build result
    alt TARGET == x86_64-unknown-linux-gnu
        ubuntulatestRunner ->> SourceCode: "cargo test --target TARGET"
        SourceCode -->> ubuntulatestRunner: Test results
    end
```

Sources: [.github/workflows/ci.yml(L13 - L30)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L13-L30)

## Documentation Job

The `doc` job handles automated documentation generation and deployment to GitHub Pages.

### Documentation Build Process

The documentation job runs independently of the CI matrix and focuses on generating comprehensive API documentation:

```mermaid
flowchart TD
DocTrigger["GitHub Event Trigger"]
DocJob["doc Jobubuntu-latest"]
Checkout["actions/checkout@v4"]
Toolchain["dtolnay/rust-toolchain@nightly"]
DocBuild["cargo doc --no-deps --all-featuresRUSTDOCFLAGS:-D rustdoc::broken_intra_doc_links-D missing-docs"]
IndexGen["Generate redirect index.htmlcargo tree | head -1 | cut -d' ' -f1"]
Condition["github.ref == default-branchAND not continue-on-error"]
Deploy["JamesIves/github-pages-deploy-action@v4Branch: gh-pagesFolder: target/docsingle-commit: true"]
Skip["Skip deployment"]
Pages["GitHub PagesDocumentation Site"]

Checkout --> DocBuild
Condition --> Deploy
Condition --> Skip
Deploy --> Pages
DocBuild --> IndexGen
DocJob --> Checkout
DocJob --> Toolchain
DocTrigger --> DocJob
IndexGen --> Condition
```

Sources: [.github/workflows/ci.yml(L32 - L56)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L32-L56)

### Documentation Configuration

The documentation build enforces strict standards through `RUSTDOCFLAGS`:

|Flag|Purpose|
| --- | --- |
|-D rustdoc::broken_intra_doc_links|Treat broken documentation links as errors|
|-D missing-docs|Require documentation for all public items|

The build generates a redirect index page using project metadata:

```
printf '<meta http-equiv="refresh" content="0;url=%s/index.html">' $(cargo tree | head -1 | cut -d' ' -f1) > target/doc/index.html
```

### Deployment Strategy

Documentation deployment occurs only on the default branch using the `JamesIves/github-pages-deploy-action@v4` action with single-commit mode to maintain a clean `gh-pages` branch history.

Sources: [.github/workflows/ci.yml(L36 - L55)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L36-L55)

## Multi-Target Architecture Support

The pipeline supports diverse target architectures to ensure compatibility across different deployment environments:

### Target Architecture Matrix

|Target|Architecture|Environment|Testing Level|
| --- | --- | --- | --- |
|x86_64-unknown-linux-gnu|x86_64|Linux with libc|Full (build + test)|
|x86_64-unknown-none|x86_64|Bare metal|Build only|
|riscv64gc-unknown-none-elf|RISC-V 64-bit|Bare metal ELF|Build only|
|aarch64-unknown-none-softfloat|ARM64|Bare metal soft-float|Build only|

The restricted testing on embedded targets reflects the procedural macro nature of the crate - the generated code needs to compile for embedded targets, but the macro itself only executes during compilation on the host.

```mermaid
flowchart TD
Source["Source CodeProcedural Macro"]
CompileTime["Compile TimeHost: x86_64-linux"]
HostTest["Host Testingx86_64-unknown-linux-gnuFull test suite"]
EmbeddedValidation["Embedded Validation"]
BareMetal["x86_64-unknown-noneBuild validation"]
RISCV["riscv64gc-unknown-none-elfBuild validation"]
ARM["aarch64-unknown-none-softfloatBuild validation"]
Runtime["RuntimeGenerated macros executeon target platforms"]

ARM --> Runtime
BareMetal --> Runtime
CompileTime --> EmbeddedValidation
CompileTime --> HostTest
EmbeddedValidation --> ARM
EmbeddedValidation --> BareMetal
EmbeddedValidation --> RISCV
HostTest --> Runtime
RISCV --> Runtime
Source --> CompileTime
```

Sources: [.github/workflows/ci.yml(L12)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L12-L12) [.github/workflows/ci.yml(L29 - L30)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/.github/workflows/ci.yml#L29-L30)