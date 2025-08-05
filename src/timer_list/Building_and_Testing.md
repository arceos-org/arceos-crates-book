# Building and Testing

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml)

This page documents the automated build and testing infrastructure for the `timer_list` crate, including the CI/CD pipeline configuration, supported target architectures, and local development commands. The information covers both automated workflows triggered by Git events and manual testing procedures for local development.

For information about the project file structure and development environment setup, see [Project Structure](/arceos-org/timer_list/4.2-project-structure).

## CI/CD Pipeline Overview

The `timer_list` crate uses GitHub Actions for continuous integration and deployment. The pipeline consists of two main jobs that ensure code quality, cross-platform compatibility, and automated documentation deployment.

### Pipeline Architecture

```mermaid
flowchart TD
A["push/pull_request"]
B["GitHub Actions Trigger"]
C["ci job"]
D["doc job"]
E["ubuntu-latest runner"]
F["ubuntu-latest runner"]
G["Matrix Strategy"]
H["rust-toolchain: nightly"]
I["4 target architectures"]
J["Format Check"]
K["Clippy Linting"]
L["Multi-target Build"]
M["Unit Tests"]
N["Documentation Build"]
O["GitHub Pages Deploy"]
P["x86_64-unknown-linux-gnu only"]
Q["gh-pages branch"]

A --> B
B --> C
B --> D
C --> E
D --> F
E --> G
E --> J
E --> K
E --> L
E --> M
F --> N
F --> O
G --> H
G --> I
M --> P
O --> Q
```

The pipeline implements a fail-fast strategy set to `false`, allowing all target builds to complete even if one fails, providing comprehensive feedback across all supported architectures.

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L1-L56)

### Build Matrix Configuration

The CI system uses a matrix strategy to test across multiple configurations simultaneously:

|Component|Value|
| --- | --- |
|Rust Toolchain|nightly|
|Runner OS|ubuntu-latest|
|Fail Fast|false|
|Target Architectures|4 cross-compilation targets|

```mermaid
flowchart TD
A["Matrix Strategy"]
B["nightly toolchain"]
C["Target Matrix"]
D["x86_64-unknown-linux-gnu"]
E["x86_64-unknown-none"]
F["riscv64gc-unknown-none-elf"]
G["aarch64-unknown-none-softfloat"]
H["rust-src component"]
I["clippy component"]
J["rustfmt component"]

A --> B
A --> C
B --> H
B --> I
B --> J
C --> D
C --> E
C --> F
C --> G
```

The pipeline installs additional Rust components (`rust-src`, `clippy`, `rustfmt`) and configures targets for each architecture in the matrix.

Sources: [.github/workflows/ci.yml(L8 - L19)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L8-L19)

## Target Architecture Support

The crate supports four target architectures, reflecting its use in embedded and operating system environments:

### Architecture Details

|Target|Purpose|Testing|
| --- | --- | --- |
|x86_64-unknown-linux-gnu|Standard Linux development|Full testing enabled|
|x86_64-unknown-none|Bare metal x86_64|Build verification only|
|riscv64gc-unknown-none-elf|RISC-V embedded systems|Build verification only|
|aarch64-unknown-none-softfloat|ARM64 embedded systems|Build verification only|

```mermaid
flowchart TD
A["timer_list Crate"]
B["Cross-Platform Support"]
C["x86_64-unknown-linux-gnu"]
D["x86_64-unknown-none"]
E["riscv64gc-unknown-none-elf"]
F["aarch64-unknown-none-softfloat"]
G["cargo test enabled"]
H["cargo build only"]
I["cargo build only"]
J["cargo build only"]
K["Unit test execution"]
L["no-std compatibility"]
M["RISC-V validation"]
N["ARM64 validation"]

A --> B
B --> C
B --> D
B --> E
B --> F
C --> G
D --> H
E --> I
F --> J
G --> K
H --> L
I --> M
J --> N
```

Unit tests execute only on `x86_64-unknown-linux-gnu` due to the testing framework requirements, while other targets verify `no-std` compatibility and cross-compilation success.

Sources: [.github/workflows/ci.yml(L12)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L12-L12) [.github/workflows/ci.yml(L29 - L30)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L29-L30)

## Build and Test Commands

The CI pipeline executes a specific sequence of commands for each target architecture. These commands can also be run locally for development purposes.

### Core CI Commands

```mermaid
sequenceDiagram
    participant DeveloperCI as "Developer/CI"
    participant RustToolchain as "Rust Toolchain"
    participant Cargo as "Cargo"
    participant TargetBinary as "Target Binary"

    DeveloperCI ->> RustToolchain: "rustc --version --verbose"
    RustToolchain -->> DeveloperCI: "Version information"
    DeveloperCI ->> Cargo: "cargo fmt --all -- --check"
    Cargo -->> DeveloperCI: "Format validation"
    DeveloperCI ->> Cargo: "cargo clippy --target TARGET --all-features"
    Cargo -->> DeveloperCI: "Lint results"
    DeveloperCI ->> Cargo: "cargo build --target TARGET --all-features"
    Cargo ->> TargetBinary: "Compile for target"
    TargetBinary -->> DeveloperCI: "Build artifacts"
    alt "x86_64-unknown-linux-gnu only"
        DeveloperCI ->> Cargo: "cargo test --target TARGET -- --nocapture"
        Cargo -->> DeveloperCI: "Test results"
    end
```

### Local Development Commands

For local development, developers can run the same commands used in CI:

|Command|Purpose|
| --- | --- |
|cargo fmt --all -- --check|Verify code formatting|
|cargo clippy --all-features|Run linter with all features|
|cargo build --all-features|Build with all features enabled|
|cargo test -- --nocapture|Run unit tests with output|

The `--all-features` flag ensures that all conditional compilation features are enabled during builds and linting.

Sources: [.github/workflows/ci.yml(L22 - L30)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L22-L30)

## Documentation Generation

The `doc` job handles automated documentation generation and deployment to GitHub Pages.

### Documentation Workflow

```mermaid
flowchart TD
A["doc job trigger"]
B["ubuntu-latest"]
C["Rust nightly setup"]
D["cargo doc build"]
E["RUSTDOCFLAGS validation"]
F["broken_intra_doc_links check"]
G["missing-docs check"]
H["target/doc generation"]
I["index.html redirect"]
J["default branch?"]
K["Deploy to gh-pages"]
L["Skip deployment"]
M["GitHub Pages"]

A --> B
B --> C
C --> D
D --> E
D --> H
E --> F
E --> G
H --> I
I --> J
J --> K
J --> L
K --> M
```

The documentation job includes strict validation using `RUSTDOCFLAGS` to ensure documentation quality by failing on broken internal links and missing documentation.

### Documentation Commands

The documentation generation process uses these key commands:

* `cargo doc --no-deps --all-features` - Generate documentation without dependencies
* `cargo tree | head -1 | cut -d' ' -f1` - Extract crate name for redirect
* Auto-generated `index.html` redirect to the main crate documentation

Sources: [.github/workflows/ci.yml(L32 - L55)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L32-L55)

## Error Handling and Permissions

The CI configuration includes specific error handling strategies:

* **Continue on Error**: Documentation builds continue on error for non-default branches and non-PR events
* **Permissions**: The `doc` job requires `contents: write` permission for GitHub Pages deployment
* **Conditional Deployment**: Pages deployment only occurs on pushes to the default branch

The `clippy` command includes the flag `-A clippy::new_without_default` to suppress specific linting warnings that are not relevant to this crate's design patterns.

Sources: [.github/workflows/ci.yml(L25)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L25-L25) [.github/workflows/ci.yml(L36 - L37)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L36-L37) [.github/workflows/ci.yml(L45)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L45-L45) [.github/workflows/ci.yml(L50)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L50-L50)