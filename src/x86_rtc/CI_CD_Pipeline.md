# CI/CD Pipeline

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml)

This document details the continuous integration and continuous deployment (CI/CD) pipeline for the x86_rtc crate, implemented through GitHub Actions. The pipeline ensures code quality, cross-platform compatibility, and automated documentation deployment.

For information about the development environment setup and local development workflows, see [Development Environment Setup](/arceos-org/x86_rtc/4.2-development-environment-setup). For details about the crate's platform requirements and target architectures, see [Platform and Architecture Requirements](/arceos-org/x86_rtc/3.2-platform-and-architecture-requirements).

## Pipeline Overview

The CI/CD pipeline is defined in [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L1-L56) and consists of two primary jobs: `ci` for code quality and testing, and `doc` for documentation generation and deployment.

```mermaid
flowchart TD
subgraph Outputs["Outputs"]
    QUALITY["Code Quality Validation"]
    PAGES["GitHub Pages Documentation"]
end
subgraph subGraph3["GitHub Actions Workflow"]
    subgraph subGraph2["doc job"]
        CHECKOUT_DOC["actions/checkout@v4"]
        TOOLCHAIN_DOC["dtolnay/rust-toolchain@nightly"]
        DOC_BUILD["cargo doc --no-deps --all-features"]
        DEPLOY["JamesIves/github-pages-deploy-action@v4"]
    end
    subgraph subGraph1["ci job"]
        MATRIX["Matrix Strategy"]
        CHECKOUT_CI["actions/checkout@v4"]
        TOOLCHAIN_CI["dtolnay/rust-toolchain@nightly"]
        FORMAT["cargo fmt --all -- --check"]
        CLIPPY["cargo clippy"]
        BUILD["cargo build"]
        TEST["cargo test"]
    end
end
subgraph subGraph0["Trigger Events"]
    PUSH["push events"]
    PR["pull_request events"]
end

BUILD --> TEST
CHECKOUT_CI --> TOOLCHAIN_CI
CHECKOUT_DOC --> TOOLCHAIN_DOC
CLIPPY --> BUILD
DEPLOY --> PAGES
DOC_BUILD --> DEPLOY
FORMAT --> CLIPPY
MATRIX --> CHECKOUT_CI
PR --> CHECKOUT_DOC
PR --> MATRIX
PUSH --> CHECKOUT_DOC
PUSH --> MATRIX
TEST --> QUALITY
TOOLCHAIN_CI --> FORMAT
TOOLCHAIN_DOC --> DOC_BUILD
```

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L1-L56)

## CI Job Configuration

The `ci` job implements a comprehensive testing strategy using a matrix approach to validate the crate across multiple target platforms.

### Matrix Strategy

|Parameter|Values|
| --- | --- |
|rust-toolchain|nightly|
|targets|x86_64-unknown-linux-gnu,x86_64-unknown-none|

The matrix configuration in [.github/workflows/ci.yml(L8 - L12)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L8-L12) ensures the crate builds and functions correctly for both standard Linux environments and bare-metal/kernel environments.

```mermaid
flowchart TD
subgraph subGraph2["Matrix Execution"]
    NIGHTLY["nightly toolchain"]
    GNU["x86_64-unknown-linux-gnu target"]
    NONE["x86_64-unknown-none target"]
    subgraph subGraph1["Job Instance 2"]
        J2_TOOLCHAIN["nightly"]
        J2_TARGET["x86_64-unknown-none"]
        J2_STEPS["All steps except Unit tests"]
    end
    subgraph subGraph0["Job Instance 1"]
        J1_TOOLCHAIN["nightly"]
        J1_TARGET["x86_64-unknown-linux-gnu"]
        J1_STEPS["All steps + Unit tests"]
    end
end

GNU --> J1_TARGET
J1_TARGET --> J1_STEPS
J1_TOOLCHAIN --> J1_STEPS
J2_TARGET --> J2_STEPS
J2_TOOLCHAIN --> J2_STEPS
NIGHTLY --> J1_TOOLCHAIN
NIGHTLY --> J2_TOOLCHAIN
NONE --> J2_TARGET
```

Sources: [.github/workflows/ci.yml(L8 - L12)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L8-L12) [.github/workflows/ci.yml(L29 - L30)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L29-L30)

### CI Job Steps

The CI job executes the following validation steps in sequence:

```mermaid
flowchart TD
CHECKOUT["actions/checkout@v4Check out repository"]
SETUP["dtolnay/rust-toolchain@nightlyInstall Rust nightly + components"]
VERSION["rustc --version --verboseVerify Rust installation"]
FORMAT["cargo fmt --all -- --checkValidate code formatting"]
CLIPPY["cargo clippy --target TARGET --all-featuresLint analysis"]
BUILD["cargo build --target TARGET --all-featuresCompilation check"]
TEST["cargo test --target TARGET -- --nocaptureUnit testing (linux-gnu only)"]

BUILD --> TEST
CHECKOUT --> SETUP
CLIPPY --> BUILD
FORMAT --> CLIPPY
SETUP --> VERSION
VERSION --> FORMAT
```

Sources: [.github/workflows/ci.yml(L13 - L30)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L13-L30)

The Rust toolchain setup [.github/workflows/ci.yml(L15 - L19)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L15-L19) includes essential components:

* `rust-src`: Source code for cross-compilation
* `clippy`: Linting tool
* `rustfmt`: Code formatting tool

Unit tests are conditionally executed only for the `x86_64-unknown-linux-gnu` target [.github/workflows/ci.yml(L29 - L30)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L29-L30) since the bare-metal target (`x86_64-unknown-none`) cannot execute standard test harnesses.

## Documentation Job

The `doc` job handles automated documentation generation and deployment to GitHub Pages.

### Documentation Build Process

```mermaid
flowchart TD
subgraph subGraph2["Documentation Generation"]
    DOC_START["Job trigger"]
    CHECKOUT["actions/checkout@v4"]
    TOOLCHAIN["dtolnay/rust-toolchain@nightly"]
    BUILD_DOC["cargo doc --no-deps --all-features"]
    INDEX_GEN["Generate index.html redirect"]
    subgraph subGraph1["Conditional Deployment"]
        BRANCH_CHECK["Check if default branch"]
        DEPLOY["JamesIves/github-pages-deploy-action@v4"]
        PAGES_OUTPUT["GitHub Pages deployment"]
    end
    subgraph subGraph0["Build Configuration"]
        FLAGS["RUSTDOCFLAGS environment"]
        BROKEN_LINKS["-D rustdoc::broken_intra_doc_links"]
        MISSING_DOCS["-D missing-docs"]
    end
end

BRANCH_CHECK --> DEPLOY
BROKEN_LINKS --> BUILD_DOC
BUILD_DOC --> INDEX_GEN
CHECKOUT --> TOOLCHAIN
DEPLOY --> PAGES_OUTPUT
DOC_START --> CHECKOUT
FLAGS --> BROKEN_LINKS
FLAGS --> MISSING_DOCS
INDEX_GEN --> BRANCH_CHECK
MISSING_DOCS --> BUILD_DOC
TOOLCHAIN --> FLAGS
```

Sources: [.github/workflows/ci.yml(L32 - L56)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L32-L56)

### Documentation Configuration

The documentation build enforces strict quality standards through `RUSTDOCFLAGS` [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L40-L40):

|Flag|Purpose|
| --- | --- |
|-D rustdoc::broken_intra_doc_links|Treat broken internal documentation links as errors|
|-D missing-docs|Require documentation for all public APIs|

The build process [.github/workflows/ci.yml(L44 - L48)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L44-L48) generates documentation using `cargo doc --no-deps --all-features` and creates an automatic redirect index page pointing to the crate's documentation root.

### Deployment Strategy

Documentation deployment follows a branch-based strategy:

```mermaid
flowchart TD
subgraph subGraph0["Branch Logic"]
    TRIGGER["Documentation build"]
    DEFAULT_CHECK["github.ref == env.default-branch"]
    DEPLOY_ACTION["Deploy to gh-pages branch"]
    CONTINUE["continue-on-error for non-default"]
    SKIP["Skip deployment"]
end

CONTINUE --> SKIP
DEFAULT_CHECK --> CONTINUE
DEFAULT_CHECK --> DEPLOY_ACTION
TRIGGER --> DEFAULT_CHECK
```

Sources: [.github/workflows/ci.yml(L45)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L45-L45) [.github/workflows/ci.yml(L49 - L55)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L49-L55)

Only builds from the default branch [.github/workflows/ci.yml(L50)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L50-L50) trigger deployment to GitHub Pages using the `JamesIves/github-pages-deploy-action@v4` action with `single-commit: true` to maintain a clean deployment history.

## Pipeline Features

### Error Handling and Robustness

The pipeline implements several error handling strategies:

* **Fail-fast disabled** [.github/workflows/ci.yml(L9)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L9-L9): Matrix jobs continue execution even if one target fails
* **Conditional execution** [.github/workflows/ci.yml(L29)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L29-L29): Unit tests only run where applicable
* **Continue-on-error** [.github/workflows/ci.yml(L45)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L45-L45): Documentation builds for non-default branches don't fail the pipeline

### Security and Permissions

The documentation job requires specific permissions [.github/workflows/ci.yml(L36 - L37)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L36-L37):

* `contents: write` for GitHub Pages deployment

### Build Optimization

Key optimization features include:

* **No-deps documentation** [.github/workflows/ci.yml(L47)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L47-L47): Faster builds by excluding dependency documentation
* **Single-commit deployment** [.github/workflows/ci.yml(L53)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L53-L53): Reduced repository size for GitHub Pages
* **All-features builds** [.github/workflows/ci.yml(L25 - L47)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L25-L47): Comprehensive feature coverage

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L1-L56)