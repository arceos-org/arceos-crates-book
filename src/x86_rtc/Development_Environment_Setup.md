# Development Environment Setup

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-org/x86_rtc/blob/1990537d/.gitignore)

## Purpose and Scope

This document provides comprehensive guidelines for setting up a development environment for the x86_rtc crate, including toolchain requirements, project structure understanding, and development best practices. It covers the essential tools, configurations, and workflows needed to contribute effectively to this hardware-level RTC driver.

For information about the automated CI/CD pipeline that validates these development practices, see [CI/CD Pipeline](/arceos-org/x86_rtc/4.1-cicd-pipeline).

## Prerequisites and Toolchain Requirements

The x86_rtc crate requires a specific Rust toolchain configuration to support its hardware-focused, no_std development model and target platforms.

### Required Rust Toolchain

The project mandates the Rust nightly toolchain with specific components and targets:

|Component|Purpose|Required|
| --- | --- | --- |
|nightlytoolchain|Access to unstable features for hardware programming|Yes|
|rust-src|Source code for cross-compilation|Yes|
|clippy|Linting and code quality analysis|Yes|
|rustfmt|Code formatting standards|Yes|
|x86_64-unknown-linux-gnu|Standard Linux development target|Yes|
|x86_64-unknown-none|Bare metal/kernel development target|Yes|

### Toolchain Installation Commands

```markdown
# Install nightly toolchain
rustup toolchain install nightly

# Add required components
rustup component add --toolchain nightly rust-src clippy rustfmt

# Add compilation targets
rustup target add --toolchain nightly x86_64-unknown-linux-gnu
rustup target add --toolchain nightly x86_64-unknown-none

# Set nightly as default (optional)
rustup default nightly
```

Sources: [.github/workflows/ci.yml(L11 - L19)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L11-L19)

### Development Toolchain Architecture

```mermaid
flowchart TD
subgraph subGraph3["Development Commands"]
    BUILD_CMD["cargo build"]
    TEST_CMD["cargo test"]
    FMT_CMD["cargo fmt"]
    CLIPPY_CMD["cargo clippy"]
    DOC_CMD["cargo doc"]
end
subgraph subGraph2["Target Platforms"]
    GNU_TARGET["x86_64-unknown-linux-gnuHosted development"]
    NONE_TARGET["x86_64-unknown-noneBare metal/kernel"]
end
subgraph subGraph1["Required Components"]
    RUST_SRC["rust-srcCross-compilation support"]
    CLIPPY["clippyCode linting"]
    RUSTFMT["rustfmtCode formatting"]
end
subgraph subGraph0["Rust Toolchain"]
    NIGHTLY["nightly toolchain"]
    COMPONENTS["Components"]
    TARGETS["Compilation Targets"]
end

CLIPPY --> CLIPPY_CMD
COMPONENTS --> CLIPPY
COMPONENTS --> RUSTFMT
COMPONENTS --> RUST_SRC
GNU_TARGET --> TEST_CMD
NIGHTLY --> COMPONENTS
NIGHTLY --> TARGETS
NONE_TARGET --> BUILD_CMD
RUSTFMT --> FMT_CMD
RUST_SRC --> BUILD_CMD
TARGETS --> GNU_TARGET
TARGETS --> NONE_TARGET
```

Sources: [.github/workflows/ci.yml(L15 - L19)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L15-L19)

## Repository Structure and Ignored Files

Understanding the project's file organization and what gets excluded from version control is essential for effective development.

### Version Control Exclusions

The `.gitignore` configuration maintains a clean repository by excluding build artifacts and development-specific files:

|Path|Description|Reason for Exclusion|
| --- | --- | --- |
|/target|Cargo build output directory|Generated artifacts, platform-specific|
|/.vscode|Visual Studio Code workspace settings|IDE-specific, personal preferences|
|.DS_Store|macOS file system metadata|Platform-specific system files|

### Development File Structure

```mermaid
flowchart TD
subgraph subGraph2["Build Process"]
    CARGO_BUILD["cargo build"]
    CARGO_TEST["cargo test"]
    CARGO_DOC["cargo doc"]
end
subgraph subGraph1["Generated/Ignored Files"]
    TARGET["/targetBuild artifacts"]
    VSCODE["/.vscodeIDE settings"]
    DSSTORE[".DS_StoremacOS metadata"]
end
subgraph subGraph0["Version Controlled Files"]
    SRC["src/lib.rsCore implementation"]
    CARGO["Cargo.tomlCrate configuration"]
    LOCK["Cargo.lockDependency versions"]
    CI["/.github/workflows/ci.ymlCI configuration"]
    README["README.mdDocumentation"]
end

CARGO --> CARGO_BUILD
CARGO_BUILD --> TARGET
CARGO_DOC --> TARGET
CARGO_TEST --> TARGET
SRC --> CARGO_BUILD
TARGET --> DSSTORE
TARGET --> VSCODE
```

Sources: [.gitignore(L1 - L4)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.gitignore#L1-L4)

## Development Workflow and Code Quality

The development workflow is designed around the CI pipeline requirements, ensuring code quality and platform compatibility before integration.

### Code Quality Pipeline

The development process follows a structured approach that mirrors the automated CI checks:

```mermaid
flowchart TD
subgraph subGraph1["Quality Gates"]
    FMT_CHECK["Formatting Check"]
    LINT_CHECK["Linting Check"]
    BUILD_CHECK["Build Check"]
    TEST_CHECK["Test Check"]
    DOC_CHECK["Documentation Check"]
end
subgraph subGraph0["Development Cycle"]
    EDIT["Edit Code"]
    FORMAT["cargo fmt --all --check"]
    LINT["cargo clippy --all-features"]
    BUILD_GNU["cargo build --target x86_64-unknown-linux-gnu"]
    BUILD_NONE["cargo build --target x86_64-unknown-none"]
    TEST["cargo test --target x86_64-unknown-linux-gnu"]
    DOC["cargo doc --no-deps --all-features"]
end

BUILD_CHECK --> TEST
BUILD_GNU --> BUILD_NONE
BUILD_NONE --> BUILD_CHECK
DOC --> DOC_CHECK
EDIT --> FORMAT
FMT_CHECK --> LINT
FORMAT --> FMT_CHECK
LINT --> LINT_CHECK
LINT_CHECK --> BUILD_GNU
TEST --> TEST_CHECK
TEST_CHECK --> DOC
```

Sources: [.github/workflows/ci.yml(L22 - L30)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L22-L30)

### Development Commands Reference

|Command|Purpose|Target Platform|CI Equivalent|
| --- | --- | --- | --- |
|cargo fmt --all -- --check|Verify code formatting|All|Line 23|
|cargo clippy --target <target> --all-features -- -D warnings|Lint analysis|Both targets|Line 25|
|cargo build --target <target> --all-features|Compilation check|Both targets|Line 27|
|cargo test --target x86_64-unknown-linux-gnu -- --nocapture|Unit testing|Linux only|Line 30|
|cargo doc --no-deps --all-features|Documentation generation|Default|Line 47|

## Local Testing Environment

### Target-Specific Testing

Testing is platform-dependent due to the hardware-specific nature of the crate:

|Test Type|Platform|Capability|Limitations|
| --- | --- | --- | --- |
|Unit Tests|x86_64-unknown-linux-gnu|Full test execution|Requires hosted environment|
|Build Tests|x86_64-unknown-none|Compilation verification only|No test execution in bare metal|
|Documentation|Default target|API documentation generation|No hardware interaction|

### Hardware Testing Considerations

Since this crate provides hardware-level RTC access, comprehensive testing requires understanding of the limitations:

```mermaid
flowchart TD
subgraph subGraph2["Hardware Interaction"]
    CMOS_ACCESS["CMOS Register AccessRequires privileged mode"]
    RTC_OPS["RTC OperationsHardware-dependent"]
end
subgraph subGraph1["Test Capabilities"]
    UNIT_TESTS["Unit TestsAPI verification"]
    BUILD_TESTS["Build TestsCompilation validation"]
    DOC_TESTS["Documentation TestsExample validation"]
end
subgraph subGraph0["Testing Environments"]
    HOSTED["Hosted Environmentx86_64-unknown-linux-gnu"]
    BARE["Bare Metal Environmentx86_64-unknown-none"]
end

BARE --> BUILD_TESTS
CMOS_ACCESS --> HOSTED
HOSTED --> BUILD_TESTS
HOSTED --> DOC_TESTS
HOSTED --> UNIT_TESTS
RTC_OPS --> HOSTED
UNIT_TESTS --> CMOS_ACCESS
UNIT_TESTS --> RTC_OPS
```

Sources: [.github/workflows/ci.yml(L28 - L30)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L28-L30)

## Documentation Development

### Local Documentation Generation

The project uses strict documentation standards enforced through `RUSTDOCFLAGS`:

```markdown
# Generate documentation with strict checks
RUSTDOCFLAGS="-D rustdoc::broken_intra_doc_links -D missing-docs" cargo doc --no-deps --all-features
```

The documentation generation process creates a redirect index for easy navigation:

```markdown
# Generate redirect index (automated in CI)
printf '<meta http-equiv="refresh" content="0;url=%s/index.html">' $(cargo tree | head -1 | cut -d' ' -f1) > target/doc/index.html
```

Sources: [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L40-L40) [.github/workflows/ci.yml(L47 - L48)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L47-L48)

## Best Practices Summary

1. **Always use nightly toolchain** - Required for hardware programming features
2. **Test on both targets** - Ensure compatibility with hosted and bare metal environments
3. **Run full quality pipeline locally** - Mirror CI checks before committing
4. **Maintain documentation standards** - All public APIs must be documented
5. **Respect platform limitations** - Understand hardware testing constraints
6. **Keep repository clean** - Respect `.gitignore` patterns for collaborative development

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.github/workflows/ci.yml#L1-L56) [.gitignore(L1 - L4)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/.gitignore#L1-L4)