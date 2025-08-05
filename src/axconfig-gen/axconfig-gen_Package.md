# axconfig-gen Package

> **Relevant source files**
> * [axconfig-gen/README.md](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/README.md)
> * [axconfig-gen/src/main.rs](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs)

This document covers the `axconfig-gen` package, which provides both a command-line tool and a Rust library for TOML-based configuration generation in the ArceOS ecosystem. The package serves as the core processing engine for configuration management, offering type-safe conversion from TOML specifications to both TOML and Rust code outputs.

For procedural macro interfaces that build on this package, see [axconfig-macros Package](/arceos-org/axconfig-gen/3-axconfig-macros-package). For specific CLI usage patterns, see [Command Line Interface](/arceos-org/axconfig-gen/2.1-command-line-interface). For programmatic API details, see [Library API](/arceos-org/axconfig-gen/2.2-library-api).

## Package Architecture

The `axconfig-gen` package operates as a dual-purpose tool, providing both standalone CLI functionality and a library API for integration into other Rust applications. The package implements a sophisticated configuration processing pipeline that handles TOML parsing, type inference, validation, and code generation.

### Core Components

```mermaid
flowchart TD
subgraph subGraph3["External Dependencies"]
    clap["clapCLI parsing"]
    toml_edit["toml_editTOML manipulation"]
end
subgraph subGraph2["Processing Core"]
    toml_parsing["TOML Parsingtoml_edit integration"]
    type_inference["Type InferenceComment-based types"]
    validation["Validationmerge() and update()"]
    code_generation["Code Generationdump() method"]
end
subgraph subGraph1["Library API"]
    Config["ConfigMain configuration container"]
    ConfigItem["ConfigItemIndividual config entries"]
    ConfigValue["ConfigValueTyped values"]
    ConfigType["ConfigTypeType system"]
    OutputFormat["OutputFormatToml | Rust"]
end
subgraph subGraph0["CLI Layer"]
    Args["Args structClap parser"]
    main_rs["main.rsCLI entry point"]
end

Args --> main_rs
Config --> ConfigItem
Config --> code_generation
Config --> toml_parsing
Config --> validation
ConfigItem --> ConfigValue
ConfigValue --> ConfigType
ConfigValue --> type_inference
code_generation --> OutputFormat
main_rs --> Config
main_rs --> ConfigValue
main_rs --> OutputFormat
main_rs --> clap
toml_parsing --> toml_edit
```

Sources: [axconfig-gen/src/main.rs(L1 - L175)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L1-L175) [axconfig-gen/README.md(L1 - L69)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/README.md#L1-L69)

### Processing Pipeline

```mermaid
flowchart TD
subgraph subGraph2["Output Generation"]
    Config_dump["Config::dump()Generate output"]
    file_output["File Output--output path"]
    stdout_output["Standard OutputDefault behavior"]
end
subgraph subGraph1["Core Processing"]
    Config_new["Config::new()Initialize container"]
    Config_from_toml["Config::from_toml()Parse TOML specs"]
    Config_merge["Config::merge()Combine specifications"]
    Config_update["Config::update()Apply old config values"]
    config_modification["CLI Read/WriteIndividual item access"]
end
subgraph subGraph0["Input Sources"]
    spec_files["Specification FilesTOML format"]
    oldconfig["Old Config FileOptional TOML"]
    cli_args["CLI Arguments--read, --write flags"]
end

Config_dump --> file_output
Config_dump --> stdout_output
Config_from_toml --> Config_merge
Config_merge --> Config_update
Config_new --> Config_merge
Config_update --> config_modification
cli_args --> config_modification
config_modification --> Config_dump
oldconfig --> Config_update
spec_files --> Config_from_toml
```

Sources: [axconfig-gen/src/main.rs(L76 - L174)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L76-L174)

## Key Capabilities

### Configuration Management

The package provides comprehensive configuration management through the `Config` struct, which serves as the central container for all configuration data. The configuration system supports:

|Feature|Description|Implementation|
| --- | --- | --- |
|Specification Loading|Load multiple TOML specification files|Config::from_toml()andConfig::merge()|
|Value Updates|Apply existing configuration values|Config::update()method|
|Individual Access|Read/write specific configuration items|config_at()andconfig_at_mut()|
|Type Safety|Enforce types through comments and inference|ConfigTypeandConfigValue|
|Global and Scoped|Support both global and table-scoped items|GLOBAL_TABLE_NAMEconstant|

### CLI Interface

The command-line interface, implemented in [axconfig-gen/src/main.rs(L5 - L40)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L5-L40) provides extensive functionality for configuration management:

```mermaid
flowchart TD
subgraph subGraph2["Output Operations"]
    format_selection["--fmt toml|rustChoose output format"]
    file_output["--output pathWrite to file"]
    backup_creation["Auto backup.old extension"]
end
subgraph subGraph0["Input Operations"]
    spec_loading["--spec filesLoad specifications"]
    oldconfig_loading["--oldconfig fileLoad existing config"]
    item_writing["--write table.key=valueSet individual items"]
end
subgraph subGraph1["Query Operations"]
    item_reading["--read table.keyGet individual items"]
    verbose_mode["--verboseDebug information"]
end

file_output --> backup_creation
format_selection --> file_output
item_reading --> verbose_mode
item_writing --> format_selection
oldconfig_loading --> format_selection
spec_loading --> format_selection
```

Sources: [axconfig-gen/src/main.rs(L5 - L40)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L5-L40) [axconfig-gen/README.md(L8 - L22)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/README.md#L8-L22)

### Type System Integration

The package implements a sophisticated type system that bridges TOML configuration with Rust type safety:

|Type Category|TOML Comment Syntax|Generated Rust Type|
| --- | --- | --- |
|Primitives|# bool,# int,# uint,# str|bool,isize,usize,&str|
|Collections|# [type]|&[type]|
|Tuples|# (type1, type2, ...)|(type1, type2, ...)|
|Inferred|No comment|Automatic from value|

### Library API

The package exposes a clean programmatic interface for integration into other Rust applications. The core workflow follows this pattern:

1. **Configuration Creation**: Initialize with `Config::new()` or `Config::from_toml()`
2. **Specification Merging**: Combine multiple sources with `Config::merge()`
3. **Value Management**: Update configurations with `Config::update()`
4. **Output Generation**: Generate code with `Config::dump(OutputFormat)`

## Integration Points

### ArceOS Build System

The package integrates with the ArceOS build system through multiple pathways:

```mermaid
flowchart TD
subgraph subGraph2["Generated Artifacts"]
    toml_configs["TOML Configurations.axconfig.toml files"]
    rust_constants["Rust Constantspub const definitions"]
    module_structure["Module Structurepub mod organization"]
end
subgraph subGraph1["axconfig-gen Processing"]
    cli_execution["CLI Executionaxconfig-gen binary"]
    library_usage["Library UsageProgrammatic API"]
    spec_processing["Specification ProcessingTOML merging"]
end
subgraph subGraph0["Build Time Integration"]
    build_scripts["Build Scriptsbuild.rs files"]
    cargo_workspace["Cargo WorkspaceMulti-crate builds"]
    env_resolution["Environment VariablesPath resolution"]
end

build_scripts --> cli_execution
cargo_workspace --> library_usage
cli_execution --> toml_configs
env_resolution --> spec_processing
library_usage --> rust_constants
module_structure --> cargo_workspace
rust_constants --> cargo_workspace
spec_processing --> module_structure
toml_configs --> cargo_workspace
```

Sources: [axconfig-gen/src/main.rs(L87 - L95)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L87-L95) [axconfig-gen/README.md(L24 - L28)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/README.md#L24-L28)

### External Tool Integration

The package supports integration with external configuration management tools through its file-based interface and backup mechanisms implemented in [axconfig-gen/src/main.rs(L155 - L170)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L155-L170) The automatic backup system ensures configuration history preservation during updates.

Sources: [axconfig-gen/src/main.rs(L155 - L170)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L155-L170) [axconfig-gen/README.md(L34 - L62)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/README.md#L34-L62)