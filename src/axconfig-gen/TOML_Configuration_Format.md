# TOML Configuration Format

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md)
> * [example-configs/defconfig.toml](https://github.com/arceos-org/axconfig-gen/blob/99357274/example-configs/defconfig.toml)

This document specifies the TOML input format used by axconfig-gen for configuration processing. It covers the structure, type annotation system, supported data types, and ArceOS-specific conventions for writing configuration files.

For information about how configurations are processed into output formats, see [Generated Output Examples](/arceos-org/axconfig-gen/4.2-generated-output-examples). For details about the CLI tool that processes these TOML files, see [Command Line Interface](/arceos-org/axconfig-gen/2.1-command-line-interface).

## Basic TOML Structure

The axconfig-gen system processes TOML files organized into a two-level hierarchy: a global table for top-level configuration items and named tables for grouped configurations.

```mermaid
flowchart TD
subgraph subGraph2["ConfigItem Internal Structure"]
    ITEM["ConfigItem"]
    VALUES["values: HashMap"]
    COMMENTS["comments: HashMap"]
end
subgraph subGraph1["Mapping to Config Structure"]
    CONFIG["Config"]
    GLOBAL_ITEMS["global: ConfigItem"]
    TABLE_ITEMS["tables: HashMap"]
end
subgraph subGraph0["TOML File Structure"]
    FILE["TOML Configuration File"]
    GLOBAL["Global Table(top-level keys)"]
    NAMED["Named Tables([section] headers)"]
end

CONFIG --> GLOBAL_ITEMS
CONFIG --> TABLE_ITEMS
FILE --> GLOBAL
FILE --> NAMED
GLOBAL --> CONFIG
GLOBAL_ITEMS --> ITEM
ITEM --> COMMENTS
ITEM --> VALUES
NAMED --> CONFIG
TABLE_ITEMS --> ITEM
```

**TOML Structure to Internal Representation Mapping**

The parser maps TOML sections directly to the internal `Config` structure, where global keys become part of the `global` `ConfigItem` and each `[section]` becomes a named table entry.

**Sources**: [README.md(L42 - L65)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L42-L65) [example-configs/defconfig.toml(L1 - L63)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/example-configs/defconfig.toml#L1-L63)

## Type Annotation System

Type information is specified through inline comments immediately following configuration values. This annotation system enables precise Rust code generation with correct type definitions.

```mermaid
flowchart TD
subgraph subGraph2["Value Integration"]
    VALUE_PARSE["Value parsing"]
    CONFIG_VALUE["ConfigValue"]
    TYPE_ASSIGN["Type assignment"]
    FINAL["ConfigValue with ConfigType"]
end
subgraph subGraph1["Type Processing"]
    TYPE_STR["Type String:'uint', 'bool', '[int]', etc."]
    TYPE_PARSE["ConfigType::from_str()"]
    TYPE_OBJ["ConfigType enum variant"]
end
subgraph subGraph0["TOML Parsing Flow"]
    INPUT["TOML Line:key = value  # type_annotation"]
    PARSE["toml_edit parsing"]
    EXTRACT["Comment Extraction"]
end

CONFIG_VALUE --> TYPE_ASSIGN
EXTRACT --> TYPE_STR
INPUT --> PARSE
PARSE --> EXTRACT
PARSE --> VALUE_PARSE
TYPE_ASSIGN --> FINAL
TYPE_OBJ --> TYPE_ASSIGN
TYPE_PARSE --> TYPE_OBJ
TYPE_STR --> TYPE_PARSE
VALUE_PARSE --> CONFIG_VALUE
```

**Type Annotation Processing Pipeline**

The system extracts type annotations from comments and converts them to `ConfigType` instances for type-safe code generation.

### Type Annotation Syntax

|Annotation|ConfigType|Rust Output Type|Example|
| --- | --- | --- | --- |
|# bool|ConfigType::Bool|bool|enabled = true  # bool|
|# int|ConfigType::Int|isize|offset = -10  # int|
|# uint|ConfigType::Uint|usize|size = 1024  # uint|
|# str|ConfigType::Str|&str|name = "test"  # str|
|# [uint]|ConfigType::Array(Uint)|&[usize]|ports = [80, 443]  # [uint]|
|# (uint, str)|ConfigType::Tuple([Uint, Str])|(usize, &str)|pair = [1, "a"]  # (uint, str)|

**Sources**: [README.md(L35 - L36)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L35-L36) [README.md(L47 - L48)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L47-L48) [example-configs/defconfig.toml(L2 - L3)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/example-configs/defconfig.toml#L2-L3)

## Supported Data Types

The type system supports primitive types, collections, and nested structures to accommodate complex ArceOS configuration requirements.

```mermaid
flowchart TD
subgraph subGraph3["ConfigType Mapping"]
    CONFIG_BOOL["ConfigType::Bool"]
    CONFIG_INT["ConfigType::Int"]
    CONFIG_UINT["ConfigType::Uint"]
    CONFIG_STR["ConfigType::Str"]
    CONFIG_ARRAY["ConfigType::Array(inner)"]
    CONFIG_TUPLE["ConfigType::Tuple(Vec)"]
end
subgraph subGraph2["Value Representations"]
    DECIMAL["Decimal: 1024"]
    HEX_INT["Hex Integer: 0x1000"]
    HEX_STR["Hex String: '0xffff_ff80_0000_0000'"]
    ARRAY_VAL["Array: [1, 2, 3]"]
    TUPLE_VAL["Tuple: [value1, value2]"]
end
subgraph subGraph1["Collection Types"]
    ARRAY["[type]homogeneous arrays"]
    TUPLE["(type1, type2, ...)heterogeneous tuples"]
end
subgraph subGraph0["Primitive Types"]
    BOOL["booltrue/false values"]
    INT["intsigned integers"]
    UINT["uintunsigned integers"]
    STR["strstring values"]
end

ARRAY --> CONFIG_ARRAY
ARRAY_VAL --> ARRAY
BOOL --> CONFIG_BOOL
DECIMAL --> UINT
HEX_INT --> UINT
HEX_STR --> UINT
INT --> CONFIG_INT
STR --> CONFIG_STR
TUPLE --> CONFIG_TUPLE
TUPLE_VAL --> TUPLE
UINT --> CONFIG_UINT
```

**ConfigType System and Value Mappings**

### Complex Type Examples

**Array Types**: Support homogeneous collections with type-safe element access:

```markdown
mmio-regions = [
    ["0xb000_0000", "0x1000_0000"],
    ["0xfe00_0000", "0xc0_0000"]
]  # [(uint, uint)]
```

**Tuple Types**: Enable heterogeneous data grouping:

```markdown
endpoint = ["192.168.1.1", 8080, true]  # (str, uint, bool)
```

**Sources**: [example-configs/defconfig.toml(L48 - L54)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/example-configs/defconfig.toml#L48-L54) [README.md(L47 - L49)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L47-L49)

## ArceOS Configuration Conventions

ArceOS configurations follow established patterns for organizing system, platform, and device specifications into logical groupings.

### Standard Configuration Sections

```mermaid
flowchart TD
subgraph subGraph4["Devices Section Details"]
    MMIO["mmio-regions: [(uint, uint)]"]
    VIRTIO["virtio-mmio-regions: [(uint, uint)]"]
    PCI["pci-ecam-base: uint"]
end
subgraph subGraph3["Platform Section Details"]
    PHYS_BASE["phys-memory-base: uint"]
    PHYS_SIZE["phys-memory-size: uint"]
    KERN_BASE["kernel-base-paddr: uint"]
    VIRT_MAP["phys-virt-offset: uint"]
end
subgraph subGraph2["Kernel Section Details"]
    STACK["task-stack-size: uint"]
    TICKS["ticks-per-sec: uint"]
end
subgraph subGraph1["Root Level Details"]
    ARCH["arch: strArchitecture identifier"]
    PLAT["plat: strPlatform identifier"]
    SMP["smp: uintCPU count"]
end
subgraph subGraph0["ArceOS Configuration Structure"]
    ROOT["Root Levelarch, plat, smp"]
    KERNEL["[kernel] SectionRuntime parameters"]
    PLATFORM["[platform] SectionMemory layout & hardware"]
    DEVICES["[devices] SectionHardware specifications"]
end

DEVICES --> MMIO
DEVICES --> PCI
DEVICES --> VIRTIO
KERNEL --> STACK
KERNEL --> TICKS
PLATFORM --> KERN_BASE
PLATFORM --> PHYS_BASE
PLATFORM --> PHYS_SIZE
PLATFORM --> VIRT_MAP
ROOT --> ARCH
ROOT --> PLAT
ROOT --> SMP
```

**ArceOS Standard Configuration Organization**

### Memory Address Conventions

ArceOS uses specific patterns for memory address specification:

|Address Type|Format|Example|Purpose|
| --- | --- | --- | --- |
|Physical addresses|Hex integers|0x20_0000|Hardware memory locations|
|Virtual addresses|Hex strings|"0xffff_ff80_0020_0000"|Kernel virtual memory|
|Memory regions|Tuple arrays|[["0xb000_0000", "0x1000_0000"]]|MMIO ranges|
|Size specifications|Hex integers|0x800_0000|Memory region sizes|

**Sources**: [example-configs/defconfig.toml(L22 - L39)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/example-configs/defconfig.toml#L22-L39) [example-configs/defconfig.toml(L48 - L62)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/example-configs/defconfig.toml#L48-L62)

## Key Naming and Value Formats

The configuration system supports flexible key naming and multiple value representation formats to accommodate diverse configuration needs.

### Key Naming Conventions

```markdown
# Standard kebab-case keys
task-stack-size = 0x1000        # uint

# Quoted keys for special characters
"one-two-three" = 456           # int

# Mixed naming in different contexts
kernel-base-paddr = 0x20_0000   # uint
"phys-memory-base" = 0          # uint
```

### Value Format Support

|Value Type|TOML Representation|Internal Processing|Output|
| --- | --- | --- | --- |
|Decimal integers|1024|Direct parsing|1024|
|Hex integers|0x1000|Hex parsing|4096|
|Hex strings|"0xffff_ff80"|String + type hint|0xffff_ff80_usize|
|Underscore separators|0x800_0000|Ignored in parsing|0x8000000|
|String literals|"x86_64"|String preservation|"x86_64"|
|Boolean values|true/false|Direct mapping|true/false|

### Type Inference Rules

When no explicit type annotation is provided:

```mermaid
flowchart TD
VALUE["TOML Value"]
CHECK_BOOL["Is boolean?"]
CHECK_INT["Is integer?"]
CHECK_STR["Is string?"]
CHECK_ARRAY["Is array?"]
INFER_BOOL["ConfigType::Bool"]
INFER_UINT["ConfigType::Uint"]
INFER_STR["ConfigType::Str"]
INFER_ARRAY["ConfigType::Array(infer element type)"]

CHECK_ARRAY --> INFER_ARRAY
CHECK_BOOL --> CHECK_INT
CHECK_BOOL --> INFER_BOOL
CHECK_INT --> CHECK_STR
CHECK_INT --> INFER_UINT
CHECK_STR --> CHECK_ARRAY
CHECK_STR --> INFER_STR
VALUE --> CHECK_BOOL
```

**Type Inference Decision Tree**

The system defaults to unsigned integers for numeric values and attempts to infer array element types recursively.

**Sources**: [README.md(L35 - L36)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/README.md#L35-L36) [example-configs/defconfig.toml(L1 - L7)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/example-configs/defconfig.toml#L1-L7) [example-configs/defconfig.toml(L28 - L32)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/example-configs/defconfig.toml#L28-L32)