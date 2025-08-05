# Project Structure and Dependencies

> **Relevant source files**
> * [.gitignore](https://github.com/arceos-org/kspin/blob/dfc0ff2c/.gitignore)
> * [Cargo.lock](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.lock)
> * [Cargo.toml](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.toml)

This document describes the organizational structure of the kspin crate, including its file layout, external dependencies, and build configuration. It covers the project's dependency management, feature flag system, and how the crate is organized to support different compilation targets.

For detailed information about the specific spinlock types and their usage, see [Spinlock Types and Public API](/arceos-org/kspin/2-spinlock-types-and-public-api). For implementation details of the core architecture, see [Core Implementation Architecture](/arceos-org/kspin/3-core-implementation-architecture).

## File Structure Overview

The kspin crate follows a minimal but well-organized structure typical of focused Rust libraries. The project consists of core source files, build configuration, and development tooling.

### Project File Organization

```mermaid
flowchart TD
subgraph subGraph2["Development Tools"]
    VSCodeDir[".vscode/IDE configuration"]
    TargetDir["target/Build artifacts (ignored)"]
end
subgraph subGraph1["Source Code (src/)"]
    LibRs["lib.rsPublic API & type aliases"]
    BaseRs["base.rsCore BaseSpinLock implementation"]
end
subgraph subGraph0["Root Directory"]
    CargoToml["Cargo.tomlProject metadata & dependencies"]
    CargoLock["Cargo.lockDependency lock file"]
    GitIgnore[".gitignoreVersion control exclusions"]
end
subgraph CI/CD["CI/CD"]
    GithubActions[".github/workflows/Automated testing & docs"]
end

CargoLock --> CargoToml
CargoToml --> BaseRs
CargoToml --> LibRs
GitIgnore --> TargetDir
GitIgnore --> VSCodeDir
LibRs --> BaseRs
```

**Sources:** [Cargo.toml(L1 - L22)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.toml#L1-L22) [.gitignore(L1 - L4)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/.gitignore#L1-L4) [Cargo.lock(L1 - L74)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.lock#L1-L74)

### Source File Responsibilities

|File|Purpose|Key Components|
| --- | --- | --- |
|src/lib.rs|Public API surface|SpinRaw,SpinNoPreempt,SpinNoIrqtype aliases|
|src/base.rs|Core implementation|BaseSpinLock,BaseSpinLockGuard,BaseGuardtrait|

The crate deliberately maintains a minimal file structure with only two source files, emphasizing simplicity and focused functionality.

**Sources:** [Cargo.toml(L1 - L22)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.toml#L1-L22)

## External Dependencies

The kspin crate has a carefully curated set of external dependencies that provide essential functionality for kernel-space synchronization and conditional compilation.

### Direct Dependencies

```mermaid
flowchart TD
subgraph subGraph2["Transitive Dependencies"]
    CrateInterface["crate_interfacev0.1.4Interface definitions"]
    ProcMacro2["proc-macro2v1.0.93Procedural macros"]
    Quote["quotev1.0.38Code generation"]
    Syn["synv2.0.96Syntax parsing"]
    UnicodeIdent["unicode-identv1.0.14Unicode support"]
end
subgraph subGraph1["Direct Dependencies"]
    CfgIf["cfg-ifv1.0.0Conditional compilation"]
    KernelGuard["kernel_guardv0.1.2Protection mechanisms"]
end
subgraph subGraph0["kspin Crate"]
    KSpin["kspinv0.1.0"]
end

CrateInterface --> ProcMacro2
CrateInterface --> Quote
CrateInterface --> Syn
KSpin --> CfgIf
KSpin --> KernelGuard
KernelGuard --> CfgIf
KernelGuard --> CrateInterface
ProcMacro2 --> UnicodeIdent
Quote --> ProcMacro2
Syn --> ProcMacro2
Syn --> Quote
Syn --> UnicodeIdent
```

**Sources:** [Cargo.toml(L19 - L21)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.toml#L19-L21) [Cargo.lock(L5 - L74)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.lock#L5-L74)

### Dependency Roles

|Crate|Version|Purpose|Usage in kspin|
| --- | --- | --- | --- |
|cfg-if|1.0.0|Conditional compilation utilities|Enables SMP vs single-core optimizations|
|kernel_guard|0.1.2|Kernel protection mechanisms|ProvidesNoOp,NoPreempt,NoPreemptIrqSaveguards|

The `kernel_guard` crate is the primary external dependency, providing the guard types that implement different protection levels. The `cfg-if` crate enables clean conditional compilation based on feature flags.

**Sources:** [Cargo.toml(L20 - L21)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.toml#L20-L21) [Cargo.lock(L23 - L30)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.lock#L23-L30)

## Feature Configuration

The kspin crate uses Cargo feature flags to enable compile-time optimization for different target environments, particularly distinguishing between single-core and multi-core systems.

### Feature Flag System

```mermaid
flowchart TD
subgraph subGraph2["Generated Code Characteristics"]
    SMPCode["SMP Implementation• AtomicBool for lock state• compare_exchange operations• Actual spinning behavior"]
    SingleCoreCode["Single-core Implementation• No lock state field• Lock always succeeds• No atomic operations"]
end
subgraph subGraph1["Compilation Modes"]
    SMPMode["SMP ModeMulti-core systems"]
    SingleCoreMode["Single-core ModeEmbedded/simple systems"]
end
subgraph subGraph0["Feature Flags"]
    SMPFeature["smpMulti-core support"]
    DefaultFeature["defaultEmpty set"]
end

DefaultFeature --> SingleCoreMode
SMPFeature --> SMPMode
SMPMode --> SMPCode
SingleCoreMode --> SingleCoreCode
```

**Sources:** [Cargo.toml(L14 - L17)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.toml#L14-L17)

### Feature Flag Details

|Feature|Default|Description|Impact|
| --- | --- | --- | --- |
|smp|No|Enable multi-core support|Adds atomic operations and actual lock state|
|default|Yes|Default feature set (empty)|Optimized for single-core environments|

The feature system allows the same codebase to generate dramatically different implementations:

* **Without `smp`**: Lock operations are compile-time no-ops, eliminating all atomic overhead
* **With `smp`**: Full atomic spinlock implementation with proper memory ordering

**Sources:** [Cargo.toml(L14 - L17)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.toml#L14-L17)

## Build System Organization

The project follows standard Rust crate conventions with specific configurations for kernel-space development and multi-platform support.

### Package Metadata Configuration

```mermaid
flowchart TD
subgraph Licensing["Licensing"]
    License["licenseGPL-3.0-or-later OR Apache-2.0 OR MulanPSL-2.0"]
end
subgraph subGraph2["Repository Links"]
    Homepage["homepageArceOS GitHub"]
    Repository["repositorykspin GitHub"]
    Documentation["documentationdocs.rs"]
end
subgraph subGraph1["Package Classification"]
    Keywords["keywords['arceos', 'synchronization', 'spinlock', 'no-irq']"]
    Categories["categories['os', 'no-std']"]
    Description["description'Spinlocks for kernel space...'"]
end
subgraph subGraph0["Package Identity"]
    Name["name = 'kspin'"]
    Version["version = '0.1.0'"]
    Edition["edition = '2021'"]
    Authors["authors = ['Yuekai Jia']"]
end

Description --> Repository
Homepage --> License
Name --> Keywords
Repository --> Documentation
Version --> Categories
```

**Sources:** [Cargo.toml(L1 - L12)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.toml#L1-L12)

### Development Environment Setup

The project includes configuration for common development tools:

|Tool|Configuration|Purpose|
| --- | --- | --- |
|Git|.gitignoreexcludes/target,/.vscode,.DS_Store|Clean repository state|
|VS Code|.vscode/directory (ignored)|IDE-specific settings|
|Cargo|Cargo.lockcommitted|Reproducible builds|

The `.gitignore` configuration ensures that build artifacts and IDE-specific files don't pollute the repository, while the committed `Cargo.lock` file ensures reproducible builds across different environments.

**Sources:** [.gitignore(L1 - L4)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/.gitignore#L1-L4) [Cargo.lock(L1 - L4)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/Cargo.lock#L1-L4)