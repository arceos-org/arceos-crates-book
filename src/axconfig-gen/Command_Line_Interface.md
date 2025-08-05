# Command Line Interface

> **Relevant source files**
> * [axconfig-gen/README.md](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/README.md)
> * [axconfig-gen/src/main.rs](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs)

This document provides comprehensive documentation for the `axconfig-gen` command-line interface (CLI) tool. The CLI enables users to process TOML configuration files, merge specifications, update configurations, and generate output in various formats for the ArceOS operating system.

For information about using axconfig-gen as a Rust library, see [Library API](/arceos-org/axconfig-gen/2.2-library-api). For compile-time configuration processing using procedural macros, see [axconfig-macros Package](/arceos-org/axconfig-gen/3-axconfig-macros-package).

## CLI Architecture Overview

The CLI tool is built around a structured argument parsing system that processes configuration specifications through a multi-stage pipeline.

**CLI Argument Processing Architecture**

```mermaid
flowchart TD
subgraph subGraph3["Output Generation"]
    CONFIG_DUMP["config.dump()"]
    FILE_WRITE["std::fs::write()"]
    STDOUT["println!()"]
end
subgraph subGraph2["Configuration Operations"]
    CONFIG_NEW["Config::new()"]
    CONFIG_FROM["Config::from_toml()"]
    CONFIG_MERGE["config.merge()"]
    CONFIG_UPDATE["config.update()"]
    CONFIG_AT["config.config_at()"]
    CONFIG_AT_MUT["config.config_at_mut()"]
end
subgraph subGraph1["Input Processing"]
    PARSE_READ["parse_config_read_arg()"]
    PARSE_WRITE["parse_config_write_arg()"]
    FILE_READ["std::fs::read_to_string()"]
end
subgraph clap::Parser["clap::Parser"]
    ARGS["Args struct"]
    SPEC["spec: Vec"]
    OLD["oldconfig: Option"]
    OUT["output: Option"]
    FMT["fmt: OutputFormat"]
    READ["read: Vec"]
    WRITE["write: Vec"]
    VERB["verbose: bool"]
end

ARGS --> FMT
ARGS --> OLD
ARGS --> OUT
ARGS --> READ
ARGS --> SPEC
ARGS --> VERB
ARGS --> WRITE
CONFIG_DUMP --> FILE_WRITE
CONFIG_DUMP --> STDOUT
CONFIG_FROM --> CONFIG_MERGE
CONFIG_FROM --> CONFIG_NEW
CONFIG_FROM --> CONFIG_UPDATE
FILE_READ --> CONFIG_FROM
FMT --> CONFIG_DUMP
OLD --> FILE_READ
OUT --> FILE_WRITE
PARSE_READ --> CONFIG_AT
PARSE_WRITE --> CONFIG_AT_MUT
READ --> PARSE_READ
SPEC --> FILE_READ
WRITE --> PARSE_WRITE
```

Sources: [axconfig-gen/src/main.rs(L5 - L40)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L5-L40) [axconfig-gen/src/main.rs(L76 - L174)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L76-L174)

## Command Line Arguments

The CLI accepts multiple arguments that control different aspects of configuration processing:

|Argument|Short|Type|Required|Description|
| --- | --- | --- | --- | --- |
|<SPEC>...|-|Vec<String>|Yes|Paths to configuration specification files|
|--oldconfig|-c|Option<String>|No|Path to existing configuration file for updates|
|--output|-o|Option<String>|No|Output file path (stdout if not specified)|
|--fmt|-f|OutputFormat|No|Output format:tomlorrust(default:toml)|
|--read|-r|Vec<String>|No|Read config items with formattable.key|
|--write|-w|Vec<String>|No|Write config items with formattable.key=value|
|--verbose|-v|bool|No|Enable verbose debug output|

### Format Specification

The `--fmt` argument accepts two values processed by a `PossibleValuesParser`:

```
toml  - Generate TOML configuration output
rust  - Generate Rust constant definitions
```

Sources: [axconfig-gen/src/main.rs(L7 - L40)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L7-L40) [axconfig-gen/README.md(L8 - L22)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/README.md#L8-L22)

## Configuration Item Addressing

Configuration items use a dot-notation addressing system implemented in parsing functions:

**Configuration Item Parsing Flow**

```

```

Sources: [axconfig-gen/src/main.rs(L42 - L62)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L42-L62) [axconfig-gen/src/main.rs(L136 - L147)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L136-L147)

## Usage Patterns

### Basic Configuration Generation

Generate a configuration file from specifications:

```
axconfig-gen spec1.toml spec2.toml -o .axconfig.toml -f toml
```

This pattern loads multiple specification files, merges them using `Config::merge()`, and outputs the result.

### Configuration Updates

Update an existing configuration with new specifications:

```
axconfig-gen defconfig.toml -c existing.toml -o updated.toml
```

The update process uses `Config::update()` which returns untouched and extra items for warning generation.

### Runtime Configuration Queries

Read specific configuration values:

```
axconfig-gen defconfig.toml -r kernel.heap_size -r features.smp
```

Write configuration values:

```
axconfig-gen defconfig.toml -w kernel.heap_size=0x100000 -w features.smp=true
```

### Rust Code Generation

Generate Rust constant definitions:

```
axconfig-gen defconfig.toml -f rust -o config.rs
```

Sources: [axconfig-gen/README.md(L24 - L28)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/README.md#L24-L28) [axconfig-gen/src/main.rs(L87 - L95)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L87-L95) [axconfig-gen/src/main.rs(L97 - L117)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L97-L117)

## Configuration Processing Pipeline

The CLI implements a sequential processing pipeline that handles merging, updating, and querying operations:

**Configuration Processing Flow**

```mermaid
flowchart TD
subgraph subGraph5["Output Generation"]
    DUMP["config.dump(args.fmt)"]
    OUTPUT_CHECK["if args.output.is_some()"]
    BACKUP["create .old backup"]
    WRITE_FILE["std::fs::write(path, output)"]
    WRITE_STDOUT["println!(output)"]
end
subgraph subGraph4["Read Operations"]
    READ_LOOP["for arg in args.read"]
    PARSE_READ_ARG["parse_config_read_arg(arg)"]
    CONFIG_AT["config.config_at(table, key)"]
    PRINT_VALUE["println!(item.value().to_toml_value())"]
    EARLY_RETURN["return if read mode"]
end
subgraph subGraph3["Write Operations"]
    WRITE_LOOP["for arg in args.write"]
    PARSE_WRITE_ARG["parse_config_write_arg(arg)"]
    CONFIG_AT_MUT["config.config_at_mut(table, key)"]
    VALUE_UPDATE["item.value_mut().update(new_value)"]
end
subgraph subGraph2["Old Config Processing"]
    OLD_CHECK["if args.oldconfig.is_some()"]
    READ_OLD["std::fs::read_to_string(oldconfig)"]
    OLD_FROM_TOML["Config::from_toml(oldconfig_toml)"]
    UPDATE["config.update(oldconfig)"]
    WARN_UNTOUCHED["warn untouched items"]
    WARN_EXTRA["warn extra items"]
end
subgraph subGraph1["Specification Loading"]
    SPEC_LOOP["for spec in args.spec"]
    READ_SPEC["std::fs::read_to_string(spec)"]
    FROM_TOML["Config::from_toml(spec_toml)"]
    MERGE["config.merge(sub_config)"]
end
subgraph Initialization["Initialization"]
    CONFIG_NEW["Config::new()"]
end

BACKUP --> WRITE_FILE
CONFIG_AT --> PRINT_VALUE
CONFIG_AT_MUT --> VALUE_UPDATE
CONFIG_NEW --> SPEC_LOOP
DUMP --> OUTPUT_CHECK
EARLY_RETURN --> DUMP
FROM_TOML --> MERGE
MERGE --> OLD_CHECK
MERGE --> SPEC_LOOP
OLD_CHECK --> READ_OLD
OLD_CHECK --> WRITE_LOOP
OLD_FROM_TOML --> UPDATE
OUTPUT_CHECK --> BACKUP
OUTPUT_CHECK --> WRITE_STDOUT
PARSE_READ_ARG --> CONFIG_AT
PARSE_WRITE_ARG --> CONFIG_AT_MUT
PRINT_VALUE --> READ_LOOP
READ_LOOP --> EARLY_RETURN
READ_LOOP --> PARSE_READ_ARG
READ_OLD --> OLD_FROM_TOML
READ_SPEC --> FROM_TOML
SPEC_LOOP --> READ_SPEC
UPDATE --> WARN_EXTRA
UPDATE --> WARN_UNTOUCHED
VALUE_UPDATE --> WRITE_LOOP
WARN_EXTRA --> WRITE_LOOP
WRITE_LOOP --> DUMP
WRITE_LOOP --> PARSE_WRITE_ARG
WRITE_LOOP --> READ_LOOP
```

Sources: [axconfig-gen/src/main.rs(L87 - L174)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L87-L174)

## Error Handling and Debugging

The CLI implements comprehensive error handling with a custom `unwrap!` macro that provides clean error messages and exits gracefully:

### Error Handling Mechanism

* File reading errors display the problematic file path
* TOML parsing errors show detailed syntax information
* Config merging conflicts are reported with item names
* Missing configuration items generate helpful error messages

### Verbose Mode

When `--verbose` is enabled, the CLI outputs debug information for:

* Configuration specification loading
* Old configuration processing
* Individual config item operations
* Output generation decisions

The verbose output uses a local `debug!` macro that conditionally prints to stderr based on the `args.verbose` flag.

Sources: [axconfig-gen/src/main.rs(L64 - L74)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L64-L74) [axconfig-gen/src/main.rs(L79 - L85)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L79-L85) [axconfig-gen/src/main.rs(L89 - L90)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L89-L90)

## File Backup System

When writing to an existing output file, the CLI automatically creates backup files to prevent data loss:

* Backup files use the `.old` extension (e.g., `config.toml` → `config.old.toml`)
* Backups are only created if the new output differs from the existing file
* The comparison prevents unnecessary writes when output is identical

This backup mechanism ensures safe configuration updates while avoiding redundant file operations.

Sources: [axconfig-gen/src/main.rs(L156 - L169)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/main.rs#L156-L169)