# Project Structure

> **Relevant source files**
> * [.gitignore](https://github.com/arceos-org/crate_interface/blob/73011a44/.gitignore)
> * [Cargo.toml](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml)

This document covers the organization and configuration of the `crate_interface` repository, including its file structure, dependencies, build setup, and development environment. For information about the actual macro implementations and testing, see [Testing](/arceos-org/crate_interface/5.1-testing) and [Macro Reference](/arceos-org/crate_interface/3-macro-reference).

## Repository Organization

The `crate_interface` project follows a standard Rust crate structure optimized for procedural macro development. The repository is intentionally minimal, focusing on a single library crate that exports three core procedural macros.

```mermaid
flowchart TD
subgraph Documentation["Documentation"]
    ReadmeMd["README.md"]
end
subgraph CI/CD["CI/CD"]
    GithubDir[".github/"]
    WorkflowsDir[".github/workflows/"]
    CiYml["ci.yml"]
end
subgraph Testing["Testing"]
    TestsDir["tests/"]
    TestFiles["Integration test files"]
end
subgraph subGraph1["Source Code"]
    SrcDir["src/"]
    LibRs["src/lib.rs"]
end
subgraph subGraph0["Configuration Files"]
    Root["crate_interface/"]
    CargoToml["Cargo.toml"]
    GitIgnore[".gitignore"]
end

GithubDir --> WorkflowsDir
Root --> CargoToml
Root --> GitIgnore
Root --> GithubDir
Root --> ReadmeMd
Root --> SrcDir
Root --> TestsDir
SrcDir --> LibRs
TestsDir --> TestFiles
WorkflowsDir --> CiYml
```

**Repository Structure Overview**

Sources: [Cargo.toml(L1 - L22)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml#L1-L22) [.gitignore(L1 - L5)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/.gitignore#L1-L5)

## Package Configuration

The project is configured as a procedural macro crate through its `Cargo.toml` manifest. The package metadata defines the crate's identity within the ArceOS ecosystem and its distribution characteristics.

```

```

**Package Configuration Details**

|Field|Value|Purpose|
| --- | --- | --- |
|name|"crate_interface"|Crate identifier for Cargo and crates.io|
|version|"0.1.4"|Semantic versioning for API compatibility|
|edition|"2021"|Rust edition for language features|
|rust-version|"1.57"|Minimum supported Rust version|
|proc-macro|true|Enables procedural macro compilation|

Sources: [Cargo.toml(L1 - L14)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml#L1-L14) [Cargo.toml(L20 - L22)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml#L20-L22)

## Dependencies Architecture

The crate relies on three essential procedural macro development dependencies, each serving a specific role in the macro compilation pipeline.

```mermaid
flowchart TD
subgraph subGraph2["Processing Pipeline"]
    TokenStream["TokenStream manipulation"]
    AstParsing["AST parsing and analysis"]
    CodeGeneration["Code generation"]
    ExternFunctions["extern function synthesis"]
end
subgraph subGraph1["Core Macro Functions"]
    DefInterface["def_interface macro"]
    ImplInterface["impl_interface macro"]
    CallInterface["call_interface! macro"]
end
subgraph subGraph0["External Dependencies"]
    ProcMacro2["proc-macro2 v1.0"]
    Quote["quote v1.0"]
    Syn["syn v2.0 with full features"]
end

AstParsing --> CallInterface
AstParsing --> DefInterface
AstParsing --> ImplInterface
CallInterface --> ExternFunctions
CodeGeneration --> CallInterface
CodeGeneration --> DefInterface
CodeGeneration --> ImplInterface
DefInterface --> ExternFunctions
ImplInterface --> ExternFunctions
ProcMacro2 --> TokenStream
Quote --> CodeGeneration
Syn --> AstParsing
TokenStream --> CallInterface
TokenStream --> DefInterface
TokenStream --> ImplInterface
```

**Dependency Functions**

|Dependency|Version|Purpose|
| --- | --- | --- |
|proc-macro2|1.0|TokenStream manipulation and span preservation|
|quote|1.0|Rust code generation with interpolation|
|syn|2.0|Rust syntax tree parsing with full feature set|

The `syn` dependency includes the `"full"` feature set to enable parsing of complete Rust syntax, including trait definitions, implementations, and method signatures required by the macro system.

Sources: [Cargo.toml(L15 - L18)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml#L15-L18)

## Build System Configuration

The crate is configured as a procedural macro library, which affects compilation behavior and usage patterns.

```mermaid
flowchart TD
subgraph subGraph2["Usage Context"]
    MacroExpansion["Compile-time macro expansion"]
    CodeGeneration["Generated extern functions"]
    CrossCrateLink["Cross-crate symbol linking"]
end
subgraph subGraph1["Compilation Output"]
    DylibFormat[".so/.dll/.dylib"]
    CompilerPlugin["Compiler plugin format"]
end
subgraph subGraph0["Build Target"]
    ProcMacroLib["proc-macro = true"]
    LibCrate["Library Crate Type"]
end

CodeGeneration --> CrossCrateLink
CompilerPlugin --> MacroExpansion
DylibFormat --> CompilerPlugin
LibCrate --> DylibFormat
MacroExpansion --> CodeGeneration
ProcMacroLib --> LibCrate
```

**Build Configuration Implications**

The `proc-macro = true` setting in `[lib]` configures the crate to:

* Compile as a dynamic library for use by the Rust compiler
* Load at compile-time during macro expansion phases
* Generate code that becomes part of dependent crates
* Enable cross-crate trait interface functionality

Sources: [Cargo.toml(L20 - L22)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml#L20-L22)

## Development Environment

The project supports multiple development workflows and target environments, as configured through the CI pipeline and package metadata.

```mermaid
flowchart TD
subgraph subGraph3["Environment Support"]
    StdEnv["std environments"]
    NoStdEnv["no-std environments"]
    EmbeddedTargets["Embedded targets"]
end
subgraph subGraph2["Development Commands"]
    CargoFmt["cargo fmt --check"]
    CargoClippy["cargo clippy"]
    CargoBuild["cargo build"]
    CargoTest["cargo test"]
    CargoDoc["cargo doc"]
end
subgraph subGraph1["Target Architectures"]
    X86Linux["x86_64-unknown-linux-gnu"]
    X86None["x86_64-unknown-none"]
    RiscV["riscv64gc-unknown-none-elf"]
    Aarch64["aarch64-unknown-none-softfloat"]
end
subgraph subGraph0["Supported Toolchains"]
    Stable["stable"]
    Beta["beta"]
    Nightly["nightly"]
    Msrv["1.57.0 (MSRV)"]
end

Aarch64 --> EmbeddedTargets
Beta --> X86None
CargoBuild --> EmbeddedTargets
CargoClippy --> NoStdEnv
CargoFmt --> StdEnv
Msrv --> Aarch64
Nightly --> RiscV
RiscV --> EmbeddedTargets
Stable --> X86Linux
X86Linux --> StdEnv
X86None --> NoStdEnv
```

**Environment Requirements**

The development environment is designed to support:

* Standard library environments for general development
* `no-std` environments for embedded and kernel development
* Multiple architectures including x86_64, RISC-V, and ARM64
* Cross-compilation for bare-metal targets

Sources: [Cargo.toml(L12 - L13)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/Cargo.toml#L12-L13) [.gitignore(L1 - L5)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/.gitignore#L1-L5)

## Version Control Configuration

The `.gitignore` configuration excludes standard Rust development artifacts and common editor files.

**Excluded Files and Directories**

|Pattern|Purpose|
| --- | --- |
|/target|Cargo build artifacts and dependencies|
|/.vscode|Visual Studio Code workspace configuration|
|.DS_Store|macOS file system metadata|
|Cargo.lock|Dependency version lockfile (appropriate for libraries)|

The exclusion of `Cargo.lock` follows Rust library conventions, allowing dependent crates to resolve their own dependency versions while maintaining compatibility with the specified version ranges.

Sources: [.gitignore(L1 - L5)&emsp;](https://github.com/arceos-org/crate_interface/blob/73011a44/.gitignore#L1-L5)