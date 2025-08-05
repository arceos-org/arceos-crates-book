# System Architecture

> **Relevant source files**
> * [Cargo.toml](https://github.com/arceos-org/axconfig-gen/blob/99357274/Cargo.toml)
> * [README.md](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md)
> * [axconfig-gen/Cargo.toml](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/Cargo.toml)
> * [axconfig-macros/Cargo.toml](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/Cargo.toml)

This document explains the architectural design of the axconfig-gen configuration system, covering how the `axconfig-gen` and `axconfig-macros` packages work together within a unified Cargo workspace. It details the component relationships, data flow patterns, and integration points that enable both CLI-based and compile-time configuration processing for ArceOS.

For specific CLI tool usage, see [Command Line Interface](/arceos-org/axconfig-gen/2.1-command-line-interface). For procedural macro implementation details, see [Macro Implementation](/arceos-org/axconfig-gen/3.2-macro-implementation).

## Workspace Organization

The axconfig-gen repository implements a dual-package architecture within a Cargo workspace that provides both standalone tooling and compile-time integration capabilities.

### Workspace Structure

```mermaid
flowchart TD
subgraph subGraph2["External Dependencies"]
    TOML_EDIT["toml_edit = '0.22'"]
    CLAP["clap = '4'"]
    PROC_MACRO["proc-macro2, quote, syn"]
end
subgraph subGraph1["axconfig-macros Package"]
    AM_LIB["src/lib.rsparse_configs!, include_configs!"]
    AM_TESTS["tests/Integration tests"]
end
subgraph subGraph0["axconfig-gen Package"]
    AG_MAIN["src/main.rsCLI entry point"]
    AG_LIB["src/lib.rsPublic API exports"]
    AG_CONFIG["src/config.rsConfig, ConfigItem"]
    AG_VALUE["src/value.rsConfigValue"]
    AG_TYPE["src/ty.rsConfigType"]
    AG_OUTPUT["src/output.rsOutputFormat"]
end
WS["Cargo Workspaceresolver = '2'"]

AG_CONFIG --> AG_OUTPUT
AG_CONFIG --> AG_TYPE
AG_CONFIG --> AG_VALUE
AG_CONFIG --> TOML_EDIT
AG_LIB --> AG_CONFIG
AG_MAIN --> AG_CONFIG
AG_MAIN --> CLAP
AM_LIB --> AG_LIB
AM_LIB --> PROC_MACRO
AM_TESTS --> AM_LIB
WS --> AG_LIB
WS --> AG_MAIN
WS --> AM_LIB
```

Sources: [Cargo.toml(L1 - L7)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/Cargo.toml#L1-L7) [axconfig-gen/Cargo.toml(L1 - L18)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/Cargo.toml#L1-L18) [axconfig-macros/Cargo.toml(L1 - L26)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/Cargo.toml#L1-L26)

## Component Architecture

The system implements a layered architecture where `axconfig-macros` depends on `axconfig-gen` for core functionality, enabling code reuse across both CLI and compile-time processing modes.

### Core Processing Components

```mermaid
flowchart TD
subgraph subGraph3["Macro Integration"]
    PARSE_CONFIGS["parse_configs!TokenStream processing"]
    INCLUDE_CONFIGS["include_configs!File path resolution"]
    PROC_MACRO_API["proc_macro2::TokenStreamquote! macro usage"]
end
subgraph subGraph2["Processing Layer"]
    TOML_PARSE["toml_edit integrationDocument parsing"]
    TYPE_INFERENCE["Type inferencefrom values and comments"]
    VALIDATION["Value validationagainst types"]
    CODE_GEN["Rust code generationpub const definitions"]
end
subgraph subGraph1["Data Model Layer"]
    CONFIG["Configglobal: ConfigTabletables: BTreeMap"]
    CONFIG_ITEM["ConfigItemvalue: ConfigValuety: Optioncomment: Option"]
    CONFIG_VALUE["ConfigValueBool, Int, Str, Array, Table"]
    CONFIG_TYPE["ConfigTypeBool, Int, UInt, Str, Tuple, Array"]
end
subgraph subGraph0["Public API Layer"]
    CONFIG_API["Config::from_toml()Config::merge()Config::update()"]
    OUTPUT_API["Config::dump(OutputFormat)"]
    CLI_API["clap::ParserArgs struct"]
end

CLI_API --> CONFIG_API
CODE_GEN --> PROC_MACRO_API
CONFIG --> CODE_GEN
CONFIG --> CONFIG_ITEM
CONFIG --> TOML_PARSE
CONFIG_API --> CONFIG
CONFIG_ITEM --> CONFIG_TYPE
CONFIG_ITEM --> CONFIG_VALUE
CONFIG_TYPE --> VALIDATION
CONFIG_VALUE --> TYPE_INFERENCE
INCLUDE_CONFIGS --> CONFIG_API
OUTPUT_API --> CONFIG
PARSE_CONFIGS --> CONFIG_API
```

Sources: [README.md(L39 - L65)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L39-L65) [README.md(L69 - L98)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L69-L98)

## Integration Patterns

The architecture supports two distinct integration patterns that share common core processing but serve different use cases in the ArceOS build pipeline.

### Dual Processing Modes

|Processing Mode|Entry Point|Input Source|Output Target|Use Case|
| --- | --- | --- | --- | --- |
|CLI Mode|axconfig-genbinary|File arguments|Generated files|Build-time configuration|
|Macro Mode|parse_configs!/include_configs!|Inline TOML / File paths|Token streams|Compile-time constants|

### Data Flow Architecture

```mermaid
flowchart TD
subgraph subGraph3["Integration Points"]
    CLI_ARGS["clap::Args--spec, --output, --fmt"]
    MACRO_INVOKE["macro invocationparse_configs!()"]
    BUILD_SYSTEM["Cargo buildintegration"]
end
subgraph subGraph2["Output Generation"]
    TOML_OUT["OutputFormat::Tomlregenerated TOML"]
    RUST_OUT["OutputFormat::Rustpub const code"]
    TOKEN_STREAM["proc_macro2::TokenStreamcompile-time expansion"]
end
subgraph subGraph1["Core Processing"]
    PARSE["toml_edit::Documentparsing"]
    CONFIG_BUILD["Config::from_toml()structure building"]
    TYPE_PROC["ConfigType inferenceand validation"]
    MERGE["Config::merge()multiple sources"]
end
subgraph subGraph0["Input Sources"]
    TOML_FILES["TOML filesdefconfig.toml"]
    TOML_STRINGS["Inline TOMLstring literals"]
    ENV_VARS["Environment variablespath resolution"]
end

CLI_ARGS --> TOML_FILES
CONFIG_BUILD --> TYPE_PROC
ENV_VARS --> TOML_FILES
MACRO_INVOKE --> TOML_STRINGS
MERGE --> RUST_OUT
MERGE --> TOKEN_STREAM
MERGE --> TOML_OUT
PARSE --> CONFIG_BUILD
RUST_OUT --> BUILD_SYSTEM
TOKEN_STREAM --> BUILD_SYSTEM
TOML_FILES --> PARSE
TOML_OUT --> BUILD_SYSTEM
TOML_STRINGS --> PARSE
TYPE_PROC --> MERGE
```

Sources: [README.md(L10 - L31)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L10-L31) [README.md(L100 - L108)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L100-L108)

## Processing Modes

The system architecture enables flexible configuration processing through two complementary approaches that share the same core data model and validation logic.

### CLI Processing Pipeline

The CLI mode implements external file processing for build-time configuration generation:

* **Input**: Multiple TOML specification files via command-line arguments
* **Processing**: File-based merging using `Config::merge()` operations
* **Output**: Generated `.axconfig.toml` or Rust constant files
* **Integration**: Build script integration via file I/O

### Macro Processing Pipeline

The macro mode implements compile-time configuration embedding:

* **Input**: Inline TOML strings or file paths in macro invocations
* **Processing**: Token stream manipulation using `proc-macro2` infrastructure
* **Output**: Rust constant definitions injected into the compilation AST
* **Integration**: Direct compile-time constant availability

### Shared Core Components

Both processing modes utilize the same underlying components:

* `Config` struct for configuration representation
* `ConfigValue` and `ConfigType` for type-safe value handling
* `toml_edit` integration for TOML parsing and manipulation
* Rust code generation logic for consistent output formatting

Sources: [axconfig-gen/Cargo.toml(L15 - L17)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/Cargo.toml#L15-L17) [axconfig-macros/Cargo.toml(L18 - L22)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/Cargo.toml#L18-L22)