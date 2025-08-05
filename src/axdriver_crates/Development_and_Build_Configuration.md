# Development and Build Configuration

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.gitignore)

This document explains the build system, continuous integration pipeline, and development workflow for the axdriver_crates repository. It covers the automated testing and validation processes, multi-target compilation support, and documentation generation that ensure code quality and maintain the driver framework across different hardware platforms.

For information about the overall architecture and design patterns, see [Architecture and Design](/arceos-org/axdriver_crates/2-architecture-and-design). For details about specific driver implementations and their usage, see the respective subsystem documentation ([Network Drivers](/arceos-org/axdriver_crates/4-network-drivers), [Block Storage Drivers](/arceos-org/axdriver_crates/5-block-storage-drivers), etc.).

## Build System Overview

The axdriver_crates workspace uses Cargo's workspace functionality to manage multiple related crates with unified build configuration. The build system supports compilation for embedded targets without standard library support, requiring careful dependency management and feature gating.

### Workspace Structure

```mermaid
flowchart TD
subgraph subGraph2["Build Steps"]
    FORMAT_CHECK["cargo fmt --checkCode Formatting"]
    CLIPPY["cargo clippyLinting"]
    BUILD["cargo buildCompilation"]
    TEST["cargo testUnit Testing"]
end
subgraph subGraph1["Build Targets"]
    X86_64_LINUX["x86_64-unknown-linux-gnuStandard Linux"]
    X86_64_NONE["x86_64-unknown-noneBare Metal x86"]
    RISCV64["riscv64gc-unknown-none-elfRISC-V Embedded"]
    AARCH64["aarch64-unknown-none-softfloatARM64 Embedded"]
end
subgraph subGraph0["Workspace Root"]
    CARGO_TOML["Cargo.tomlWorkspace Configuration"]
    CI_YML[".github/workflows/ci.ymlBuild Pipeline"]
    GITIGNORE[".gitignoreBuild Artifacts"]
end

BUILD --> AARCH64
BUILD --> RISCV64
BUILD --> X86_64_LINUX
BUILD --> X86_64_NONE
CARGO_TOML --> BUILD
CI_YML --> BUILD
CI_YML --> CLIPPY
CI_YML --> FORMAT_CHECK
CI_YML --> TEST
```

The build system validates code across multiple architectures to ensure compatibility with various embedded and desktop platforms that ArceOS supports.

**Sources:** [.github/workflows/ci.yml(L1 - L58)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.github/workflows/ci.yml#L1-L58) [.gitignore(L1 - L5)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.gitignore#L1-L5)

## Continuous Integration Pipeline

The CI pipeline implements a comprehensive validation strategy using GitHub Actions, ensuring code quality and cross-platform compatibility for all commits and pull requests.

### CI Job Matrix Configuration

```mermaid
flowchart TD
subgraph subGraph2["Validation Steps"]
    SETUP["Setup Toolchainrust-src, clippy, rustfmt"]
    FORMAT["Format Checkcargo fmt --all -- --check"]
    LINT["Lintingcargo clippy --all-features"]
    COMPILE["Buildcargo build --all-features"]
    UNIT_TEST["Unit Testscargo test (Linux only)"]
end
subgraph subGraph1["CI Matrix"]
    TOOLCHAIN["nightlyRust Toolchain"]
    TARGET_1["x86_64-unknown-linux-gnu"]
    TARGET_2["x86_64-unknown-none"]
    TARGET_3["riscv64gc-unknown-none-elf"]
    TARGET_4["aarch64-unknown-none-softfloat"]
end
subgraph subGraph0["CI Trigger Events"]
    PUSH["git push"]
    PR["Pull Request"]
end

COMPILE --> UNIT_TEST
FORMAT --> LINT
LINT --> COMPILE
PR --> TOOLCHAIN
PUSH --> TOOLCHAIN
SETUP --> FORMAT
TOOLCHAIN --> TARGET_1
TOOLCHAIN --> TARGET_2
TOOLCHAIN --> TARGET_3
TOOLCHAIN --> TARGET_4
```

The pipeline uses the `fail-fast: false` strategy to ensure all target platforms are tested even if one fails, providing comprehensive feedback for cross-platform issues.

**Sources:** [.github/workflows/ci.yml(L6 - L30)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.github/workflows/ci.yml#L6-L30)

### Documentation Pipeline

The documentation build process generates and deploys API documentation automatically for the main branch:

```mermaid
flowchart TD
subgraph subGraph2["Documentation Features"]
    INDEX_PAGE["--enable-index-page"]
    BROKEN_LINKS["-D rustdoc::broken_intra_doc_links"]
    MISSING_DOCS["-D missing-docs"]
end
subgraph subGraph1["Documented Crates"]
    BASE["axdriver_base"]
    BLOCK["axdriver_block"]
    NET["axdriver_net"]
    DISPLAY["axdriver_display"]
    PCI["axdriver_pci"]
    VIRTIO["axdriver_virtio"]
end
subgraph subGraph0["Documentation Job"]
    DOC_TRIGGER["Main Branch Push"]
    DOC_BUILD["rustdoc Generation"]
    DOC_DEPLOY["GitHub Pages Deploy"]
end

DOC_BUILD --> BASE
DOC_BUILD --> BLOCK
DOC_BUILD --> BROKEN_LINKS
DOC_BUILD --> DISPLAY
DOC_BUILD --> DOC_DEPLOY
DOC_BUILD --> INDEX_PAGE
DOC_BUILD --> MISSING_DOCS
DOC_BUILD --> NET
DOC_BUILD --> PCI
DOC_BUILD --> VIRTIO
DOC_TRIGGER --> DOC_BUILD
```

The documentation pipeline enforces strict documentation requirements with `-D missing-docs` and validates all internal links to maintain high-quality API documentation.

**Sources:** [.github/workflows/ci.yml(L32 - L58)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.github/workflows/ci.yml#L32-L58)

## Target Platform Support

The framework supports multiple target platforms with different characteristics and constraints:

|Target Platform|Purpose|Standard Library|Use Case|
| --- | --- | --- | --- |
|x86_64-unknown-linux-gnu|Development/Testing|Full|Unit tests, development|
|x86_64-unknown-none|Bare Metal x86|No-std|PC-based embedded systems|
|riscv64gc-unknown-none-elf|RISC-V Embedded|No-std|RISC-V microcontrollers|
|aarch64-unknown-none-softfloat|ARM64 Embedded|No-std|ARM-based embedded systems|

### Compilation Configuration

The build system uses `--all-features` to ensure maximum compatibility testing across all optional features. Target-specific compilation is managed through:

* **Toolchain Requirements**: Nightly Rust with `rust-src` component for no-std targets
* **Component Dependencies**: `clippy` and `rustfmt` for code quality validation
* **Feature Testing**: All features enabled during compilation to catch integration issues

**Sources:** [.github/workflows/ci.yml(L11 - L12)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.github/workflows/ci.yml#L11-L12) [.github/workflows/ci.yml(L17 - L19)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.github/workflows/ci.yml#L17-L19) [.github/workflows/ci.yml(L25 - L27)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.github/workflows/ci.yml#L25-L27)

## Development Workflow

### Code Quality Gates

The development workflow enforces strict quality standards through automated checks:

```mermaid
flowchart TD
subgraph subGraph2["Quality Requirements"]
    NO_FORMAT_ERRORS["No Format Violations"]
    NO_LINT_ERRORS["No Clippy Warnings(except new_without_default)"]
    ALL_TARGETS_BUILD["All Targets Compile"]
    TESTS_PASS["Unit Tests Pass"]
end
subgraph subGraph1["Automated Validation"]
    FORMAT_GATE["Format Checkcargo fmt --all -- --check"]
    CLIPPY_GATE["Lint Checkcargo clippy -A clippy::new_without_default"]
    BUILD_GATE["Cross-Platform Build4 target platforms"]
    TEST_GATE["Unit TestsLinux target only"]
end
subgraph subGraph0["Developer Workflow"]
    CODE_CHANGE["Code Changes"]
    LOCAL_TEST["Local Testing"]
    COMMIT["Git Commit"]
    PUSH["Git Push/PR"]
end

BUILD_GATE --> ALL_TARGETS_BUILD
BUILD_GATE --> TEST_GATE
CLIPPY_GATE --> BUILD_GATE
CLIPPY_GATE --> NO_LINT_ERRORS
CODE_CHANGE --> LOCAL_TEST
COMMIT --> PUSH
FORMAT_GATE --> CLIPPY_GATE
FORMAT_GATE --> NO_FORMAT_ERRORS
LOCAL_TEST --> COMMIT
PUSH --> FORMAT_GATE
TEST_GATE --> TESTS_PASS
```

### Clippy Configuration

The CI pipeline uses a relaxed clippy configuration with `-A clippy::new_without_default` to accommodate embedded development patterns where implementing `Default` may not be appropriate for hardware drivers.

**Sources:** [.github/workflows/ci.yml(L22 - L30)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.github/workflows/ci.yml#L22-L30)

## Build Artifacts and Caching

### Ignored Files and Directories

The repository maintains a minimal `.gitignore` configuration focusing on essential build artifacts:

```markdown
/target         # Cargo build output
/.vscode        # VSCode workspace files  
.DS_Store       # macOS system files
Cargo.lock      # Lock file (workspace)
```

The `Cargo.lock` exclusion is intentional for library crates, allowing consumers to use their preferred dependency versions while maintaining reproducible builds during development.

**Sources:** [.gitignore(L1 - L5)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.gitignore#L1-L5)

### Documentation Deployment

Documentation deployment uses a single-commit strategy to GitHub Pages, minimizing repository size while maintaining complete API documentation history through the `gh-pages` branch.

**Sources:** [.github/workflows/ci.yml(L51 - L57)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.github/workflows/ci.yml#L51-L57)

## Feature Configuration Strategy

The build system uses `--all-features` compilation to ensure comprehensive testing of all optional functionality. This approach validates:

* **Feature Compatibility**: All feature combinations compile successfully
* **Cross-Platform Features**: Features work across all supported target platforms
* **Integration Testing**: Optional components integrate properly with core functionality

This strategy supports the modular architecture where different embedded systems can selectively enable only required driver components while maintaining compatibility guarantees.

**Sources:** [.github/workflows/ci.yml(L25)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.github/workflows/ci.yml#L25-L25) [.github/workflows/ci.yml(L27)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.github/workflows/ci.yml#L27-L27) [.github/workflows/ci.yml(L47 - L50)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/.github/workflows/ci.yml#L47-L50)