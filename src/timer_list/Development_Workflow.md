# Development Workflow

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-org/timer_list/blob/4fa2875f/.gitignore)

This document provides guidance for contributors working on the `timer_list` crate. It covers local development setup, build processes, testing procedures, and the automated CI/CD pipeline. For detailed API documentation, see [Core API Reference](/arceos-org/timer_list/2-core-api-reference). For usage examples and integration patterns, see [Usage Guide and Examples](/arceos-org/timer_list/3-usage-guide-and-examples).

## Local Development Environment

The `timer_list` crate requires the Rust nightly toolchain with specific components and target architectures. The development environment must support cross-compilation for embedded and bare-metal targets.

### Required Toolchain Components

```mermaid
flowchart TD
A["rust-toolchain: nightly"]
B["rust-src component"]
C["clippy component"]
D["rustfmt component"]
E["Target Architectures"]
F["x86_64-unknown-linux-gnu"]
G["x86_64-unknown-none"]
H["riscv64gc-unknown-none-elf"]
I["aarch64-unknown-none-softfloat"]

A --> B
A --> C
A --> D
A --> E
E --> F
E --> G
E --> H
E --> I
```

**Toolchain Setup**

The project requires `rustc nightly` with cross-compilation support for multiple architectures. Install the required components:

```
rustup toolchain install nightly
rustup component add rust-src clippy rustfmt --toolchain nightly
rustup target add x86_64-unknown-none riscv64gc-unknown-none-elf aarch64-unknown-none-softfloat --toolchain nightly
```

Sources: [.github/workflows/ci.yml(L15 - L19)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L15-L19)

## Build and Development Commands

### Core Development Workflow

```mermaid
flowchart TD
A["cargo fmt --all -- --check"]
B["cargo clippy --target TARGET --all-features"]
C["cargo build --target TARGET --all-features"]
D["cargo test --target x86_64-unknown-linux-gnu"]
E["cargo doc --no-deps --all-features"]
F["Format Check"]
G["Lint Analysis"]
H["Cross-Platform Build"]
I["Unit Testing"]
J["Documentation Generation"]

A --> B
B --> C
C --> D
D --> E
F --> A
G --> B
H --> C
I --> D
J --> E
```

**Format Checking**

```
cargo fmt --all -- --check
```

**Lint Analysis**

```
cargo clippy --target <TARGET> --all-features -- -A clippy::new_without_default
```

**Building for Specific Targets**

```
cargo build --target <TARGET> --all-features
```

**Running Tests**

```
cargo test --target x86_64-unknown-linux-gnu -- --nocapture
```

Sources: [.github/workflows/ci.yml(L22 - L30)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L22-L30)

## Multi-Target Architecture Support

The crate supports four distinct target architectures, each serving different deployment scenarios:

|Target Architecture|Use Case|Testing Support|
| --- | --- | --- |
|x86_64-unknown-linux-gnu|Standard Linux development|Full unit testing|
|x86_64-unknown-none|Bare-metal x86_64 systems|Build-only|
|riscv64gc-unknown-none-elf|RISC-V embedded systems|Build-only|
|aarch64-unknown-none-softfloat|ARM64 embedded systems|Build-only|

```mermaid
flowchart TD
A["timer_list crate"]
B["x86_64-unknown-linux-gnu"]
C["x86_64-unknown-none"]
D["riscv64gc-unknown-none-elf"]
E["aarch64-unknown-none-softfloat"]
F["Linux DevelopmentUnit Testing"]
G["Bare-metal x86_64Build Verification"]
H["RISC-V EmbeddedBuild Verification"]
I["ARM64 EmbeddedBuild Verification"]

A --> B
A --> C
A --> D
A --> E
B --> F
C --> G
D --> H
E --> I
```

Unit tests execute only on `x86_64-unknown-linux-gnu` due to the `no-std` nature of other targets and the lack of standard library support for test execution.

Sources: [.github/workflows/ci.yml(L12)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L12-L12) [.github/workflows/ci.yml(L29 - L30)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L29-L30)

## Continuous Integration Pipeline

### CI Job Matrix Strategy

```mermaid
flowchart TD
subgraph subGraph1["Documentation Pipeline"]
    C["Documentation Job"]
    H["cargo doc build"]
    I["GitHub Pages Deploy"]
end
subgraph subGraph0["CI Job Matrix"]
    B["CI Job"]
    D["Format Check"]
    E["Clippy Analysis"]
    F["Multi-Target Build"]
    G["Unit Tests"]
end
A["GitHub Push/PR Trigger"]
J["x86_64-unknown-linux-gnu build"]
K["x86_64-unknown-none build"]
L["riscv64gc-unknown-none-elf build"]
M["aarch64-unknown-none-softfloat build"]
N["x86_64-unknown-linux-gnu tests only"]

A --> B
A --> C
B --> D
B --> E
B --> F
B --> G
C --> H
C --> I
F --> J
F --> K
F --> L
F --> M
G --> N
```

**CI Job Execution Steps:**

1. **Environment Setup**: Ubuntu latest with nightly Rust toolchain
2. **Format Verification**: `cargo fmt --all -- --check`
3. **Lint Analysis**: `cargo clippy` with custom configuration
4. **Cross-Compilation**: Build for all four target architectures
5. **Test Execution**: Unit tests on Linux target only

**Documentation Job Features:**

* Builds documentation with strict link checking
* Enforces complete documentation coverage
* Deploys to GitHub Pages on default branch pushes
* Generates redirect index page automatically

Sources: [.github/workflows/ci.yml(L6 - L31)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L6-L31) [.github/workflows/ci.yml(L32 - L55)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L32-L55)

## Documentation Standards

### Documentation Pipeline Configuration

```mermaid
flowchart TD
A["RUSTDOCFLAGS Environment"]
B["-D rustdoc::broken_intra_doc_links"]
C["-D missing-docs"]
D["cargo doc --no-deps --all-features"]
E["Documentation Build"]
F["Index Page Generation"]
G["GitHub Pages Deployment"]
H["Default Branch Push"]
I["Pull Request"]
J["Build Only"]

A --> B
A --> C
D --> E
E --> F
F --> G
H --> G
I --> J
```

The documentation system enforces strict standards:

* **Broken Link Detection**: All intra-doc links must resolve correctly
* **Complete Coverage**: Missing documentation generates build failures
* **Dependency Exclusion**: Only project documentation is generated (`--no-deps`)
* **Feature Complete**: All features are documented (`--all-features`)

The system automatically generates an index redirect page using the crate name extracted from `cargo tree` output.

Sources: [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L40-L40) [.github/workflows/ci.yml(L47 - L48)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L47-L48)

## Repository Structure and Ignored Files

### Git Configuration

```mermaid
flowchart TD
A["Repository Root"]
B["Source Files"]
C["Ignored Artifacts"]
D["src/"]
E["Cargo.toml"]
F[".github/workflows/"]
G["/target"]
H["/.vscode"]
I[".DS_Store"]
J["Cargo.lock"]

A --> B
A --> C
B --> D
B --> E
B --> F
C --> G
C --> H
C --> I
C --> J
```

**Ignored Files and Directories:**

* `/target`: Rust build artifacts and compiled binaries
* `/.vscode`: Visual Studio Code workspace settings
* `.DS_Store`: macOS filesystem metadata files
* `Cargo.lock`: Dependency lock file (excluded for library crates)

The `Cargo.lock` exclusion indicates this crate follows Rust library conventions, allowing downstream projects to resolve their own dependency versions.

Sources: [.gitignore(L1 - L4)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.gitignore#L1-L4)

## Contributing Guidelines

### Code Quality Requirements

All contributions must pass the complete CI pipeline:

1. **Format Compliance**: Code must conform to `rustfmt` standards
2. **Lint Compliance**: Must pass `clippy` analysis with project-specific exceptions
3. **Cross-Platform Compatibility**: Must build successfully on all four target architectures
4. **Test Coverage**: New functionality requires corresponding unit tests
5. **Documentation**: All public APIs must include comprehensive documentation

### Development Branch Strategy

The project uses a straightforward branching model:

* **Default Branch**: Primary development and release branch
* **Feature Branches**: Individual contributions via pull requests
* **GitHub Pages**: Automatic deployment from default branch

Documentation deployment occurs automatically on default branch pushes, ensuring the published documentation remains current with the latest code changes.

Sources: [.github/workflows/ci.yml(L3)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L3-L3) [.github/workflows/ci.yml(L50)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/.github/workflows/ci.yml#L50-L50)