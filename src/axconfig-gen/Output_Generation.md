# Output Generation

> **Relevant source files**
> * [axconfig-gen/src/output.rs](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/output.rs)
> * [axconfig-gen/src/tests.rs](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/tests.rs)
> * [axconfig-gen/src/value.rs](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs)

This section covers the output generation system in axconfig-gen, which converts parsed configuration structures into formatted output strings. The system supports generating both TOML files and Rust source code from configuration data, handling proper formatting, type annotations, and structural organization.

For information about the core data structures being formatted, see [Core Data Structures](/arceos-org/axconfig-gen/2.2.1-core-data-structures). For details about the type system that drives code generation, see [Type System](/arceos-org/axconfig-gen/2.2.2-type-system).

## Purpose and Scope

The output generation system serves as the final stage of the configuration processing pipeline, transforming internal `Config`, `ConfigItem`, and `ConfigValue` structures into human-readable and machine-consumable formats. It handles:

* **Format Selection**: Supporting TOML and Rust code output formats
* **Structural Organization**: Managing tables, modules, and hierarchical output
* **Type-Aware Generation**: Converting values according to their configured types
* **Formatting and Indentation**: Ensuring readable, properly formatted output
* **Comment Preservation**: Maintaining documentation and type annotations

## Output Format Architecture

The system centers around the `OutputFormat` enum and `Output` writer, which coordinate the transformation process.

**Output Format Selection**

```mermaid
flowchart TD
subgraph subGraph2["Generation Methods"]
    TABLE_BEGIN["table_begin()"]
    TABLE_END["table_end()"]
    WRITE_ITEM["write_item()"]
    PRINT_LINES["print_lines()"]
end
subgraph subGraph1["Writer System"]
    OUTPUT["Output"]
    WRITER_STATE["writer statefmt: OutputFormatindent: usizeresult: String"]
end
subgraph subGraph0["Format Selection"]
    OF["OutputFormat"]
    TOML_FMT["OutputFormat::Toml"]
    RUST_FMT["OutputFormat::Rust"]
end
TOML_OUT["[table]key = value # type"]
RUST_OUT["pub mod table {pub const KEY: Type = value;}"]

OF --> OUTPUT
OF --> RUST_FMT
OF --> TOML_FMT
OUTPUT --> PRINT_LINES
OUTPUT --> TABLE_BEGIN
OUTPUT --> TABLE_END
OUTPUT --> WRITER_STATE
OUTPUT --> WRITE_ITEM
RUST_FMT --> RUST_OUT
TOML_FMT --> TOML_OUT
```

Sources: [axconfig-gen/src/output.rs(L3 - L32)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/output.rs#L3-L32) [axconfig-gen/src/output.rs(L34 - L48)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/output.rs#L34-L48)

## Value Conversion Pipeline

The conversion from internal values to output strings follows a type-aware transformation process.

**Value to Output Conversion Flow**

```

```

Sources: [axconfig-gen/src/value.rs(L92 - L102)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L92-L102) [axconfig-gen/src/value.rs(L226 - L241)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L226-L241) [axconfig-gen/src/value.rs(L243 - L288)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L243-L288)

## Output Writer Implementation

The `Output` struct manages the formatting state and provides methods for building structured output.

### Core Writer Methods

|Method|Purpose|TOML Behavior|Rust Behavior|
| --- | --- | --- | --- |
|table_begin()|Start a configuration section|Writes[table-name]header|Createspub mod table_name {|
|table_end()|End a configuration section|No action needed|Writes closing}|
|write_item()|Output a key-value pair|key = value # type|pub const KEY: Type = value;|
|print_lines()|Handle multi-line comments|Preserves#comments|Converts to///doc comments|

Sources: [axconfig-gen/src/output.rs(L71 - L93)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/output.rs#L71-L93) [axconfig-gen/src/output.rs(L95 - L136)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/output.rs#L95-L136)

### Formatting and Indentation

The system maintains proper indentation for nested structures:

```mermaid
flowchart TD
subgraph subGraph1["Rust Module Nesting"]
    MOD_BEGIN["table_begin() → indent += 4"]
    MOD_END["table_end() → indent -= 4"]
    CONST_WRITE["write_item() uses current indent"]
end
subgraph subGraph0["Indentation Management"]
    INDENT_STATE["indent: usize"]
    PRINTLN_FMT["println_fmt()"]
    SPACES["format!('{:indent$}', '')"]
end

CONST_WRITE --> INDENT_STATE
INDENT_STATE --> PRINTLN_FMT
MOD_BEGIN --> INDENT_STATE
MOD_END --> INDENT_STATE
PRINTLN_FMT --> SPACES
```

Sources: [axconfig-gen/src/output.rs(L54 - L60)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/output.rs#L54-L60) [axconfig-gen/src/output.rs(L82 - L84)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/output.rs#L82-L84) [axconfig-gen/src/output.rs(L89 - L92)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/output.rs#L89-L92)

## Type-Specific Value Generation

Different value types require specialized conversion logic for both TOML and Rust output formats.

### Primitive Types

|Value Type|TOML Output|Rust Output|
| --- | --- | --- |
|Boolean|true,false|true,false|
|Integer|42,0xdead_beef|42,0xdead_beef|
|String|"hello"|"hello"|
|Numeric String|"0xff"|0xff(converted to number)|

Sources: [axconfig-gen/src/value.rs(L228 - L230)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L228-L230) [axconfig-gen/src/value.rs(L245 - L255)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L245-L255)

### Collection Types

**Array Conversion Logic**

```mermaid
flowchart TD
subgraph subGraph2["Rust Output"]
    SIMPLE_RUST["&[elem1, elem2, elem3]"]
    NESTED_RUST["&[elem1,elem2,]"]
end
subgraph subGraph1["TOML Output"]
    SIMPLE_TOML["[elem1, elem2, elem3]"]
    NESTED_TOML["[elem1,elem2]"]
end
subgraph subGraph0["Array Processing"]
    ARRAY_VAL["Value::Array"]
    CHECK_NESTED["contains nested arrays?"]
    ELEMENTS["convert each element"]
end

ARRAY_VAL --> CHECK_NESTED
CHECK_NESTED --> ELEMENTS
CHECK_NESTED --> NESTED_RUST
CHECK_NESTED --> NESTED_TOML
CHECK_NESTED --> SIMPLE_RUST
CHECK_NESTED --> SIMPLE_TOML
ELEMENTS --> NESTED_RUST
ELEMENTS --> NESTED_TOML
ELEMENTS --> SIMPLE_RUST
ELEMENTS --> SIMPLE_TOML
```

Sources: [axconfig-gen/src/value.rs(L231 - L240)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L231-L240) [axconfig-gen/src/value.rs(L267 - L285)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L267-L285)

### Tuple vs Array Distinction

The system distinguishes between homogeneous arrays and heterogeneous tuples:

|Configuration|Type|TOML Output|Rust Output|
| --- | --- | --- | --- |
|[1, 2, 3]|[uint]|[1, 2, 3]|&[1, 2, 3]|
|[1, "a", true]|(uint, str, bool)|[1, "a", true]|(1, "a", true)|

Sources: [axconfig-gen/src/value.rs(L256 - L265)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L256-L265) [axconfig-gen/src/value.rs(L267 - L285)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L267-L285)

## Name Transformation

The system applies consistent naming conventions when converting between formats:

**Identifier Transformation Rules**

```mermaid
flowchart TD
subgraph subGraph0["TOML to Rust Naming"]
    TOML_KEY["TOML key: 'some-config-key'"]
    MOD_TRANSFORM["mod_name(): replace '-' with '_'"]
    CONST_TRANSFORM["const_name(): UPPERCASE + replace '-' with '_'"]
    RUST_MOD["Rust module: 'some_config_key'"]
    RUST_CONST["Rust constant: 'SOME_CONFIG_KEY'"]
end

CONST_TRANSFORM --> RUST_CONST
MOD_TRANSFORM --> RUST_MOD
TOML_KEY --> CONST_TRANSFORM
TOML_KEY --> MOD_TRANSFORM
```

Sources: [axconfig-gen/src/output.rs(L138 - L144)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/output.rs#L138-L144)

## Error Handling in Output Generation

The system provides comprehensive error handling for output generation scenarios:

|Error Condition|When It Occurs|Error Type|
| --- | --- | --- |
|Unknown type for key|Type inference fails and no explicit type|ConfigErr::Other|
|Value type mismatch|Value doesn't match expected type|ConfigErr::ValueTypeMismatch|
|Array length mismatch|Tuple and array length differ|ConfigErr::ValueTypeMismatch|

Sources: [axconfig-gen/src/output.rs(L120 - L125)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/output.rs#L120-L125) [axconfig-gen/src/value.rs(L257 - L259)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L257-L259)

The output generation system provides a robust foundation for transforming configuration data into multiple target formats while preserving type information and maintaining readable formatting conventions.