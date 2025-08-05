# Overview

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md)

## Purpose and Scope

This document provides an overview of the `axconfig-gen` repository, a TOML-based configuration generation system designed for ArceOS. The system provides two primary interfaces: a command-line tool (`axconfig-gen`) for build-time configuration generation and procedural macros (`axconfig-macros`) for compile-time configuration embedding. The system transforms TOML configuration specifications into either TOML output files or Rust constant definitions with proper type annotations.

For detailed CLI usage patterns, see [Command Line Interface](/arceos-org/axconfig-gen/2.1-command-line-interface). For comprehensive macro usage, see [Macro Usage Patterns](/arceos-org/axconfig-gen/3.1-macro-usage-patterns). For configuration format specifications, see [TOML Configuration Format](/arceos-org/axconfig-gen/4.1-toml-configuration-format).

## System Architecture

The repository implements a dual-interface configuration system organized as a Cargo workspace with two main crates that share core processing logic.

### Component Architecture

```mermaid
flowchart TD
subgraph subGraph3["External Dependencies"]
    CLAP["clap (CLI parsing)"]
    TOML_EDIT["toml_edit (TOML manipulation)"]
    PROC_MACRO["proc-macro2, quote, syn"]
end
subgraph subGraph2["Cargo Workspace"]
    subgraph subGraph1["axconfig-macros Crate"]
        PARSE["parse_configs! macro"]
        INCLUDE["include_configs! macro"]
        MACLIB["Macro Implementation (lib.rs)"]
    end
    subgraph subGraph0["axconfig-gen Crate"]
        CLI["CLI Tool (main.rs)"]
        LIB["Library API (lib.rs)"]
        CONFIG["Config Structures"]
        VALUE["ConfigValue Types"]
        OUTPUT["Output Generation"]
    end
end

CLI --> CLAP
CLI --> CONFIG
CLI --> OUTPUT
CONFIG --> TOML_EDIT
INCLUDE --> LIB
LIB --> CONFIG
LIB --> VALUE
MACLIB --> LIB
MACLIB --> PROC_MACRO
PARSE --> LIB
```

**Sources:** [README.md(L1 - L109)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L1-L109) [Cargo.toml workspace structure implied]

### Core Processing Pipeline

```mermaid
flowchart TD
subgraph subGraph2["Output Generation"]
    TOML_OUT["TOML Output (.axconfig.toml)"]
    RUST_OUT["Rust Constants (pub const)"]
    COMPILE_TIME["Compile-time Embedding"]
end
subgraph subGraph1["Core Processing (axconfig-gen)"]
    PARSER["Config::from_toml()"]
    VALIDATOR["Config validation & merging"]
    TYPESYS["ConfigType inference"]
    FORMATTER["OutputFormat selection"]
end
subgraph subGraph0["Input Sources"]
    TOML_FILES["TOML Files (.toml)"]
    TOML_STRINGS["Inline TOML Strings"]
    TYPE_COMMENTS["Type Annotations (# comments)"]
end

FORMATTER --> COMPILE_TIME
FORMATTER --> RUST_OUT
FORMATTER --> TOML_OUT
PARSER --> VALIDATOR
TOML_FILES --> PARSER
TOML_STRINGS --> PARSER
TYPESYS --> FORMATTER
TYPE_COMMENTS --> TYPESYS
VALIDATOR --> TYPESYS
```

**Sources:** [README.md(L39 - L65)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L39-L65) [README.md(L69 - L98)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L69-L98)

## Usage Modes

The system operates in two distinct modes, each targeting different use cases in the ArceOS build pipeline.

|Mode|Interface|Input Source|Output Target|Use Case|
| --- | --- | --- | --- | --- |
|CLI Mode|axconfig-gencommand|File paths (<SPEC>...)|File output (-o,-f)|Build-time generation|
|Macro Mode|parse_configs!,include_configs!|Inline strings or file paths|Token stream|Compile-time embedding|

### CLI Mode Operation

The CLI tool processes multiple TOML specification files and generates output files in either TOML or Rust format.

```mermaid
flowchart TD
subgraph Outputs["Outputs"]
    TOML_FILE[".axconfig.toml"]
    RUST_FILE["constants.rs"]
end
subgraph Processing["Processing"]
    MERGE["Config merging"]
    VALIDATE["Type validation"]
    GENERATE["Output generation"]
end
subgraph subGraph0["CLI Arguments"]
    SPEC["... (input files)"]
    OUTPUT["-o(output path)"]
    FORMAT["-f  (toml|rust)"]
    OLDCONFIG["-c  (merging)"]
end

FORMAT --> GENERATE
GENERATE --> RUST_FILE
GENERATE --> TOML_FILE
MERGE --> VALIDATE
OLDCONFIG --> MERGE
OUTPUT --> GENERATE
SPEC --> MERGE
VALIDATE --> GENERATE
```

**Sources:** [README.md(L8 - L31)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L8-L31)

### Macro Mode Operation

Procedural macros embed configuration processing directly into the compilation process, generating constants at compile time.

```mermaid
flowchart TD
subgraph subGraph2["Generated Constants"]
    GLOBAL_CONST["pub const GLOBAL_VAR"]
    MODULE_CONST["pub mod module_name"]
    TYPED_CONST["Typed constants"]
end
subgraph subGraph1["Compile-time Processing"]
    TOKEN_PARSE["TokenStream parsing"]
    CONFIG_PROC["Config processing"]
    CODE_GEN["Rust code generation"]
end
subgraph subGraph0["Macro Invocations"]
    PARSE_INLINE["parse_configs!(toml_string)"]
    INCLUDE_FILE["include_configs!(file_path)"]
    INCLUDE_ENV["include_configs!(path_env = VAR)"]
end

CODE_GEN --> GLOBAL_CONST
CODE_GEN --> MODULE_CONST
CODE_GEN --> TYPED_CONST
CONFIG_PROC --> CODE_GEN
INCLUDE_ENV --> TOKEN_PARSE
INCLUDE_FILE --> TOKEN_PARSE
PARSE_INLINE --> TOKEN_PARSE
TOKEN_PARSE --> CONFIG_PROC
```

**Sources:** [README.md(L67 - L108)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L67-L108)

## Type System and Output Generation

The system implements a sophisticated type inference and annotation system that enables generation of properly typed Rust constants.

### Type Annotation System

Configuration values support type annotations through TOML comments, enabling precise Rust type generation:

* `bool` - Boolean values
* `int` - Signed integers (`isize`)
* `uint` - Unsigned integers (`usize`)
* `str` - String literals (`&str`)
* `[type]` - Arrays of specified type
* `(type1, type2, ...)` - Tuples with mixed types

### Configuration Structure Mapping

```mermaid
flowchart TD
subgraph subGraph1["Rust Output Structure"]
    PUB_CONST["pub const GLOBAL_ITEMS"]
    PUB_MOD["pub mod section_name"]
    NESTED_CONST["pub const SECTION_ITEMS"]
    TYPED_VALUES["Properly typed values"]
end
subgraph subGraph0["TOML Structure"]
    GLOBAL_TABLE["Global Table (root level)"]
    NAMED_TABLES["Named Tables ([section])"]
    CONFIG_ITEMS["Key-Value Pairs"]
    TYPE_ANNOTATIONS["Unsupported markdown: heading"]
end

CONFIG_ITEMS --> NESTED_CONST
GLOBAL_TABLE --> PUB_CONST
NAMED_TABLES --> PUB_MOD
PUB_MOD --> NESTED_CONST
TYPE_ANNOTATIONS --> TYPED_VALUES
```

**Sources:** [README.md(L35 - L36)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L35-L36) [README.md(L42 - L64)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L42-L64) [README.md(L89 - L97)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L89-L97)

## Integration with ArceOS

The configuration system integrates into the ArceOS build process through multiple pathways, providing flexible configuration management for the operating system's components.

```mermaid
flowchart TD
subgraph subGraph3["ArceOS Build"]
    CARGO_BUILD["Cargo Build Process"]
    ARCEOS_KERNEL["ArceOS Kernel"]
end
subgraph subGraph2["Generated Artifacts"]
    AXCONFIG[".axconfig.toml"]
    RUST_CONSTS["Rust Constants"]
    EMBEDDED_CONFIG["Compile-time Config"]
end
subgraph subGraph1["Configuration Processing"]
    CLI_GEN["axconfig-gen CLI"]
    MACRO_PROC["axconfig-macros processing"]
end
subgraph Development["Development"]
    DEV_CONFIG["defconfig.toml"]
    BUILD_SCRIPTS["Build Scripts"]
    RUST_SOURCES["Rust Source Files"]
end

AXCONFIG --> CARGO_BUILD
BUILD_SCRIPTS --> CLI_GEN
CARGO_BUILD --> ARCEOS_KERNEL
CLI_GEN --> AXCONFIG
CLI_GEN --> RUST_CONSTS
DEV_CONFIG --> CLI_GEN
DEV_CONFIG --> MACRO_PROC
EMBEDDED_CONFIG --> CARGO_BUILD
MACRO_PROC --> EMBEDDED_CONFIG
RUST_CONSTS --> CARGO_BUILD
RUST_SOURCES --> MACRO_PROC
```

**Sources:** [README.md(L27 - L33)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L27-L33) [README.md(L102 - L108)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L102-L108)

This architecture enables ArceOS to maintain consistent configuration across build-time generation and compile-time embedding, supporting both static configuration files and dynamic configuration processing during compilation.