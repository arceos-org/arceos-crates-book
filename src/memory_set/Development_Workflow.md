# Development Workflow

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-org/memory_set/blob/73b51e2b/.gitignore)

This document covers the continuous integration setup, testing strategies, and development environment configuration for contributors to the memory_set crate. It explains how code quality is maintained, how testing is automated, and what tools developers need to work effectively with the codebase.

For information about project dependencies and build configuration, see [Dependencies and Configuration](/arceos-org/memory_set/4.1-dependencies-and-configuration). For usage examples and testing patterns, see [Advanced Examples and Testing](/arceos-org/memory_set/3.2-advanced-examples-and-testing).

## CI Pipeline Overview

The memory_set crate uses GitHub Actions for continuous integration, with a comprehensive pipeline that ensures code quality across multiple Rust targets and maintains documentation.

### CI Workflow Structure

```mermaid
flowchart TD
PR["Push / Pull Request"]
CIJob["ci job"]
DocJob["doc job"]
Matrix["Matrix Strategy"]
T1["x86_64-unknown-linux-gnu"]
T2["x86_64-unknown-none"]
T3["riscv64gc-unknown-none-elf"]
T4["aarch64-unknown-none-softfloat"]
Steps1["Format → Clippy → Build → Test"]
Steps2["Format → Clippy → Build"]
Steps3["Format → Clippy → Build"]
Steps4["Format → Clippy → Build"]
DocBuild["cargo doc --no-deps --all-features"]
GHPages["Deploy to gh-pages"]
Success["CI Success"]

CIJob --> Matrix
DocBuild --> GHPages
DocJob --> DocBuild
GHPages --> Success
Matrix --> T1
Matrix --> T2
Matrix --> T3
Matrix --> T4
PR --> CIJob
PR --> DocJob
Steps1 --> Success
Steps2 --> Success
Steps3 --> Success
Steps4 --> Success
T1 --> Steps1
T2 --> Steps2
T3 --> Steps3
T4 --> Steps4
```

**Sources:** [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L1-L56)

### Target Platform Matrix

The CI pipeline tests across multiple Rust targets to ensure compatibility with different architectures and environments:

|Target|Purpose|Testing Level|
| --- | --- | --- |
|x86_64-unknown-linux-gnu|Standard Linux development|Full (includes unit tests)|
|x86_64-unknown-none|Bare metal x86_64|Build and lint only|
|riscv64gc-unknown-none-elf|RISC-V bare metal|Build and lint only|
|aarch64-unknown-none-softfloat|ARM64 bare metal|Build and lint only|

**Sources:** [.github/workflows/ci.yml(L11 - L12)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L11-L12)

## Code Quality Pipeline

### Quality Check Sequence

```mermaid
sequenceDiagram
    participant Developer as "Developer"
    participant GitHubActions as "GitHub Actions"
    participant RustCompiler as "Rust Compiler"
    participant ClippyLinter as "Clippy Linter"
    participant rustfmt as "rustfmt"

    Developer ->> GitHubActions: "git push / PR"
    GitHubActions ->> rustfmt: "cargo fmt --all -- --check"
    rustfmt -->> GitHubActions: "Format check result"
    GitHubActions ->> ClippyLinter: "cargo clippy --target TARGET --all-features"
    ClippyLinter ->> RustCompiler: "Run analysis"
    RustCompiler -->> ClippyLinter: "Lint results"
    ClippyLinter -->> GitHubActions: "Clippy results (-A clippy::new_without_default)"
    GitHubActions ->> RustCompiler: "cargo build --target TARGET --all-features"
    RustCompiler -->> GitHubActions: "Build result"
    alt Target is x86_64-unknown-linux-gnu
        GitHubActions ->> RustCompiler: "cargo test --target TARGET -- --nocapture"
        RustCompiler -->> GitHubActions: "Test results"
    end
    GitHubActions -->> Developer: "CI Status"
```

**Sources:** [.github/workflows/ci.yml(L22 - L30)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L22-L30)

### Code Quality Tools Configuration

The CI pipeline enforces several code quality standards:

* **Format Checking**: Uses `cargo fmt --all -- --check` to ensure consistent code formatting
* **Linting**: Runs `cargo clippy` with custom configuration allowing `clippy::new_without_default`
* **Compilation**: Verifies code builds successfully across all target platforms
* **Testing**: Executes unit tests on the primary Linux target with `--nocapture` flag

**Sources:** [.github/workflows/ci.yml(L22 - L30)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L22-L30)

## Testing Strategy

### Test Execution Environment

```mermaid
flowchart TD
subgraph subGraph1["Test Execution"]
    OnlyLinux["Tests run only on x86_64-unknown-linux-gnu"]
    NoCapture["--nocapture flag enabled"]
    UnitTests["Unit tests from lib.rs and modules"]
end
subgraph subGraph0["Test Environment"]
    Ubuntu["ubuntu-latest runner"]
    Nightly["Rust nightly toolchain"]
    Components["rust-src + clippy + rustfmt"]
end
subgraph subGraph2["Cross-Platform Validation"]
    BuildOnly["Build-only testing on bare metal targets"]
    LintOnly["Clippy linting on all targets"]
    FormatAll["Format checking across all code"]
end

BuildOnly --> LintOnly
Components --> OnlyLinux
LintOnly --> FormatAll
Nightly --> OnlyLinux
OnlyLinux --> NoCapture
OnlyLinux --> UnitTests
Ubuntu --> OnlyLinux
```

**Sources:** [.github/workflows/ci.yml(L28 - L30)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L28-L30) [.github/workflows/ci.yml(L11 - L19)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L11-L19)

### Testing Limitations and Strategy

The testing strategy reflects the embedded/systems programming nature of the memory_set crate:

* **Full Testing**: Only performed on `x86_64-unknown-linux-gnu` where standard library features are available
* **Build Verification**: All bare metal targets are compiled to ensure cross-platform compatibility
* **Mock Testing**: The crate includes `MockBackend` implementation for testing without real page tables

**Sources:** [.github/workflows/ci.yml(L29 - L30)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L29-L30)

## Documentation Workflow

### Documentation Build and Deployment

```mermaid
flowchart TD
subgraph Deployment["Deployment"]
    DefaultBranch["Is default branch?"]
    GHPages["Deploy to gh-pages"]
    SkipDeploy["Skip deployment"]
end
subgraph subGraph1["Quality Checks"]
    BrokenLinks["-D rustdoc::broken_intra_doc_links"]
    MissingDocs["-D missing-docs"]
end
subgraph subGraph0["Doc Generation"]
    DocBuild["cargo doc --no-deps --all-features"]
    DocFlags["RUSTDOCFLAGS checks"]
    IndexGen["Generate redirect index.html"]
end

BrokenLinks --> IndexGen
DefaultBranch --> GHPages
DefaultBranch --> SkipDeploy
DocBuild --> DocFlags
DocFlags --> BrokenLinks
DocFlags --> MissingDocs
IndexGen --> DefaultBranch
MissingDocs --> IndexGen
```

**Sources:** [.github/workflows/ci.yml(L32 - L56)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L32-L56)

### Documentation Quality Standards

The documentation workflow enforces strict quality standards:

* **Broken Link Detection**: `-D rustdoc::broken_intra_doc_links` fails build on broken documentation links
* **Missing Documentation**: `-D missing-docs` requires all public APIs to have documentation
* **Automatic Deployment**: Documentation is automatically deployed to GitHub Pages on default branch updates

**Sources:** [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L40-L40) [.github/workflows/ci.yml(L44 - L55)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L44-L55)

## Development Environment Setup

### Required Toolchain Components

For local development, contributors need:

```markdown
# Install nightly Rust toolchain
rustup toolchain install nightly

# Add required components
rustup component add rust-src clippy rustfmt

# Add target platforms for cross-compilation testing
rustup target add x86_64-unknown-none
rustup target add riscv64gc-unknown-none-elf  
rustup target add aarch64-unknown-none-softfloat
```

**Sources:** [.github/workflows/ci.yml(L15 - L19)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L15-L19)

### Local Development Commands

To replicate CI checks locally:

```markdown
# Format check
cargo fmt --all -- --check

# Linting  
cargo clippy --all-features -- -A clippy::new_without_default

# Build for different targets
cargo build --target x86_64-unknown-linux-gnu --all-features
cargo build --target x86_64-unknown-none --all-features

# Run tests
cargo test -- --nocapture

# Generate documentation
cargo doc --no-deps --all-features
```

**Sources:** [.github/workflows/ci.yml(L22 - L30)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L22-L30) [.github/workflows/ci.yml(L47)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.github/workflows/ci.yml#L47-L47)

### Git Configuration

The project maintains a standard `.gitignore` configuration:

* `/target` - Rust build artifacts
* `/.vscode` - IDE configuration files
* `.DS_Store` - macOS system files
* `Cargo.lock` - Dependency lock file (excluded for libraries)

**Sources:** [.gitignore(L1 - L5)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/.gitignore#L1-L5)