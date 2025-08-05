# Macro Implementation

> **Relevant source files**
> * [axconfig-macros/src/lib.rs](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/src/lib.rs)
> * [axconfig-macros/tests/example_config.rs](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/tests/example_config.rs)

This document covers the technical implementation details of the procedural macros in the `axconfig-macros` crate, including their integration with the core `axconfig-gen` library. It explains how the macros process TOML configurations at compile time and generate Rust code using the proc-macro infrastructure.

For information about how to use these macros in practice, see [Macro Usage Patterns](/arceos-org/axconfig-gen/3.1-macro-usage-patterns). For details about the underlying configuration processing logic, see [Library API](/arceos-org/axconfig-gen/2.2-library-api).

## Procedural Macro Architecture

The `axconfig-macros` crate implements two procedural macros that provide compile-time TOML configuration processing by leveraging the core functionality from `axconfig-gen`. The architecture separates parsing and code generation concerns from the macro expansion logic.

```mermaid
flowchart TD
subgraph subGraph3["File System Operations"]
    ENV_VAR_RESOLUTION["Environment VariableResolution"]
    FILE_READING["std::fs::read_to_stringTOML file loading"]
    PATH_RESOLUTION["CARGO_MANIFEST_DIRPath resolution"]
end
subgraph subGraph2["Proc-Macro Infrastructure"]
    TOKEN_STREAM["TokenStreamInput parsing"]
    SYN_PARSE["syn::parseSyntax tree parsing"]
    QUOTE_GEN["quote!Code generation"]
    COMPILE_ERROR["Error::to_compile_errorCompilation errors"]
end
subgraph subGraph1["Core Processing (axconfig-gen)"]
    CONFIG_FROM_TOML["Config::from_tomlTOML parsing"]
    CONFIG_DUMP["Config::dumpOutputFormat::Rust"]
    RUST_CODE_GEN["Rust Code Generationpub const, pub mod"]
end
subgraph subGraph0["Compile-Time Processing"]
    USER_CODE["User Rust Codeparse_configs! / include_configs!"]
    MACRO_INVOKE["Macro InvocationTokenStream input"]
    MACRO_IMPL["Macro Implementationlib.rs"]
end

COMPILE_ERROR --> USER_CODE
CONFIG_DUMP --> RUST_CODE_GEN
CONFIG_FROM_TOML --> CONFIG_DUMP
ENV_VAR_RESOLUTION --> PATH_RESOLUTION
FILE_READING --> CONFIG_FROM_TOML
MACRO_IMPL --> COMPILE_ERROR
MACRO_IMPL --> ENV_VAR_RESOLUTION
MACRO_IMPL --> FILE_READING
MACRO_IMPL --> TOKEN_STREAM
MACRO_INVOKE --> MACRO_IMPL
QUOTE_GEN --> USER_CODE
RUST_CODE_GEN --> QUOTE_GEN
SYN_PARSE --> CONFIG_FROM_TOML
TOKEN_STREAM --> SYN_PARSE
USER_CODE --> MACRO_INVOKE
```

**Sources:** [axconfig-macros/src/lib.rs(L1 - L144)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/src/lib.rs#L1-L144)

## Macro Implementation Details

### parse_configs! Macro

The `parse_configs!` macro processes inline TOML strings and expands them into Rust code at compile time. It handles both regular compilation and nightly compiler features for enhanced expression expansion.

```mermaid
flowchart TD
INPUT_TOKENS["TokenStreamTOML string literal"]
NIGHTLY_EXPAND["proc_macro::expand_exprNightly feature"]
PARSE_LITSTR["parse_macro_input!as LitStr"]
EXTRACT_VALUE["LitStr::valueExtract TOML content"]
CONFIG_PARSE["Config::from_tomlParse TOML structure"]
CONFIG_DUMP["Config::dumpOutputFormat::Rust"]
TOKEN_PARSE["code.parseString to TokenStream"]
ERROR_HANDLING["compiler_errorLexError handling"]
FINAL_TOKENS["TokenStreamGenerated Rust code"]

CONFIG_DUMP --> ERROR_HANDLING
CONFIG_DUMP --> TOKEN_PARSE
CONFIG_PARSE --> CONFIG_DUMP
CONFIG_PARSE --> ERROR_HANDLING
ERROR_HANDLING --> FINAL_TOKENS
EXTRACT_VALUE --> CONFIG_PARSE
INPUT_TOKENS --> NIGHTLY_EXPAND
INPUT_TOKENS --> PARSE_LITSTR
NIGHTLY_EXPAND --> PARSE_LITSTR
PARSE_LITSTR --> EXTRACT_VALUE
TOKEN_PARSE --> ERROR_HANDLING
TOKEN_PARSE --> FINAL_TOKENS
```

The implementation includes conditional compilation for nightly features:

|Feature|Functionality|Implementation|
| --- | --- | --- |
|Nightly|Enhanced expression expansion|config_toml.expand_expr()|
|Stable|Standard literal parsing|parse_macro_input!(config_toml as LitStr)|
|Error Handling|Compilation error generation|compiler_error()function|

**Sources:** [axconfig-macros/src/lib.rs(L22 - L41)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/src/lib.rs#L22-L41)

### include_configs! Macro

The `include_configs!` macro supports three different path specification methods and handles file system operations with comprehensive error handling.

```mermaid
flowchart TD
subgraph subGraph2["File Processing"]
    FILE_READ["std::fs::read_to_stringTOML file reading"]
    PARSE_CONFIGS_CALL["parse_configs!Recursive macro call"]
    QUOTE_MACRO["quote!Token generation"]
end
subgraph subGraph1["Path Resolution"]
    ENV_VAR_LOOKUP["std::env::varEnvironment variable lookup"]
    FALLBACK_LOGIC["Fallback pathhandling"]
    CARGO_MANIFEST["CARGO_MANIFEST_DIRRoot directory resolution"]
    PATH_JOIN["std::path::Path::joinFull path construction"]
end
subgraph subGraph0["Argument Parsing"]
    ARGS_INPUT["TokenStreamMacro arguments"]
    PARSE_ARGS["IncludeConfigsArgs::parseCustom argument parser"]
    PATH_DIRECT["Path variantDirect file path"]
    PATH_ENV["PathEnv variantEnvironment variable"]
    PATH_ENV_FALLBACK["PathEnvFallback variantEnv var + fallback"]
end

ARGS_INPUT --> PARSE_ARGS
CARGO_MANIFEST --> PATH_JOIN
ENV_VAR_LOOKUP --> FALLBACK_LOGIC
FALLBACK_LOGIC --> CARGO_MANIFEST
FILE_READ --> PARSE_CONFIGS_CALL
PARSE_ARGS --> PATH_DIRECT
PARSE_ARGS --> PATH_ENV
PARSE_ARGS --> PATH_ENV_FALLBACK
PARSE_CONFIGS_CALL --> QUOTE_MACRO
PATH_DIRECT --> CARGO_MANIFEST
PATH_ENV --> ENV_VAR_LOOKUP
PATH_ENV_FALLBACK --> ENV_VAR_LOOKUP
PATH_JOIN --> FILE_READ
```

**Sources:** [axconfig-macros/src/lib.rs(L58 - L87)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/src/lib.rs#L58-L87)

## Argument Parsing Implementation

The `IncludeConfigsArgs` enum and its `Parse` implementation handle the complex argument parsing for the `include_configs!` macro with proper error reporting.

```mermaid
stateDiagram-v2
[*] --> CheckFirstToken
CheckFirstToken --> DirectPath : LitStr
CheckFirstToken --> ParseParameters : Ident
DirectPath --> [*] : Path(LitStr)
ParseParameters --> ReadIdent
ReadIdent --> CheckEquals : ident
CheckEquals --> ReadString : =
ReadString --> ProcessParameter : LitStr
ProcessParameter --> SetPathEnv : "path_env"
ProcessParameter --> SetFallback : "fallback"
ProcessParameter --> Error : unknown parameter
SetPathEnv --> CheckComma
SetFallback --> CheckComma
CheckComma --> ReadIdent : more tokens
CheckComma --> ValidateResult : end of input
ValidateResult --> PathEnvOnly : env only
ValidateResult --> PathEnvWithFallback : env + fallback
ValidateResult --> Error : invalid combination
PathEnvOnly --> [*] : PathEnv(LitStr)
PathEnvWithFallback --> [*] : PathEnvFallback(LitStr, LitStr)
Error --> [*] : ParseError
```

The parsing logic handles these parameter combinations:

|Syntax|Enum Variant|Behavior|
| --- | --- | --- |
|"path/to/file.toml"|Path(LitStr)|Direct file path|
|path_env = "ENV_VAR"|PathEnv(LitStr)|Environment variable only|
|path_env = "ENV_VAR", fallback = "default.toml"|PathEnvFallback(LitStr, LitStr)|Environment variable with fallback|

**Sources:** [axconfig-macros/src/lib.rs(L89 - L143)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/src/lib.rs#L89-L143)

## Error Handling and Compilation

The macro implementation includes comprehensive error handling that generates meaningful compilation errors for various failure scenarios.

### Compiler Error Generation

The `compiler_error` function provides a centralized mechanism for generating compilation errors that integrate properly with the Rust compiler's diagnostic system.

```mermaid
flowchart TD
subgraph subGraph2["Compiler Integration"]
    RUST_COMPILER["Rust CompilerError reporting"]
    BUILD_FAILURE["Build FailureClear error messages"]
end
subgraph subGraph1["Error Processing"]
    COMPILER_ERROR_FN["compiler_errorError::new_spanned"]
    TO_COMPILE_ERROR["Error::to_compile_errorTokenStream generation"]
    ERROR_SPAN["Span informationSource location"]
end
subgraph subGraph0["Error Sources"]
    TOML_PARSE_ERROR["TOML Parse ErrorConfig::from_toml"]
    FILE_READ_ERROR["File Read Errorstd::fs::read_to_string"]
    ENV_VAR_ERROR["Environment Variable Errorstd::env::var"]
    LEX_ERROR["Lexical ErrorTokenStream parsing"]
    SYNTAX_ERROR["Syntax ErrorArgument parsing"]
end

COMPILER_ERROR_FN --> ERROR_SPAN
COMPILER_ERROR_FN --> TO_COMPILE_ERROR
ENV_VAR_ERROR --> COMPILER_ERROR_FN
FILE_READ_ERROR --> COMPILER_ERROR_FN
LEX_ERROR --> COMPILER_ERROR_FN
RUST_COMPILER --> BUILD_FAILURE
SYNTAX_ERROR --> COMPILER_ERROR_FN
TOML_PARSE_ERROR --> COMPILER_ERROR_FN
TO_COMPILE_ERROR --> RUST_COMPILER
```

**Sources:** [axconfig-macros/src/lib.rs(L12 - L14)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/src/lib.rs#L12-L14) [axconfig-macros/src/lib.rs(L36 - L40)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/src/lib.rs#L36-L40) [axconfig-macros/src/lib.rs(L63 - L67)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/src/lib.rs#L63-L67) [axconfig-macros/src/lib.rs(L79 - L81)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/src/lib.rs#L79-L81)

## Integration with Build System

The macros integrate with Cargo's build system through environment variable resolution and dependency tracking that ensures proper rebuilds when configuration files change.

### Build-Time Path Resolution

The implementation uses `CARGO_MANIFEST_DIR` to resolve relative paths consistently across different build environments:

|Environment Variable|Purpose|Usage|
| --- | --- | --- |
|CARGO_MANIFEST_DIR|Project root directory|Path resolution base|
|User-defined env vars|Config file paths|Dynamic path specification|

### Dependency Tracking

The file-based operations in `include_configs!` automatically create implicit dependencies that Cargo uses for rebuild detection:

```mermaid
flowchart TD
subgraph subGraph2["Build Outputs"]
    COMPILED_BINARY["Compiled binarywith embedded config"]
    BUILD_CACHE["Build cacheDependency information"]
end
subgraph subGraph1["Build Process"]
    MACRO_EXPANSION["Macro expansionFile read operation"]
    CARGO_TRACKING["Cargo dependency trackingImplicit file dependency"]
    REBUILD_DETECTION["Rebuild detectionFile modification time"]
end
subgraph subGraph0["Source Files"]
    RUST_SOURCE["Rust source filewith include_configs!"]
    TOML_CONFIG["TOML config filereferenced by macro"]
end

CARGO_TRACKING --> BUILD_CACHE
CARGO_TRACKING --> REBUILD_DETECTION
MACRO_EXPANSION --> CARGO_TRACKING
REBUILD_DETECTION --> COMPILED_BINARY
RUST_SOURCE --> MACRO_EXPANSION
TOML_CONFIG --> MACRO_EXPANSION
TOML_CONFIG --> REBUILD_DETECTION
```

**Sources:** [axconfig-macros/src/lib.rs(L76 - L77)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/src/lib.rs#L76-L77) [axconfig-macros/tests/example_config.rs(L4 - L5)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/tests/example_config.rs#L4-L5)

## Testing Integration

The macro implementation includes comprehensive testing that validates both the generated code and the macro expansion process itself.

The test structure demonstrates proper integration between the macros and expected outputs:

```mermaid
flowchart TD
subgraph subGraph2["Test Execution"]
    MOD_CMP_MACRO["mod_cmp! macroModule comparison"]
    ASSERT_STATEMENTS["assert_eq! callsValue validation"]
    TEST_FUNCTIONS["test functionsinclude and parse tests"]
end
subgraph subGraph1["Test Data"]
    DEFCONFIG_TOML["defconfig.tomlTest input"]
    OUTPUT_RS["output.rsExpected generated code"]
    INCLUDE_STR["include_str!String inclusion"]
end
subgraph subGraph0["Test Configuration"]
    TEST_FILE["example_config.rsTest definitions"]
    CONFIG_MODULE["config moduleinclude_configs! usage"]
    CONFIG2_MODULE["config2 moduleparse_configs! usage"]
    EXPECTED_MODULE["config_expect moduleExpected output"]
end

ASSERT_STATEMENTS --> CONFIG2_MODULE
ASSERT_STATEMENTS --> CONFIG_MODULE
ASSERT_STATEMENTS --> EXPECTED_MODULE
CONFIG2_MODULE --> INCLUDE_STR
CONFIG_MODULE --> DEFCONFIG_TOML
EXPECTED_MODULE --> OUTPUT_RS
INCLUDE_STR --> DEFCONFIG_TOML
MOD_CMP_MACRO --> ASSERT_STATEMENTS
TEST_FUNCTIONS --> MOD_CMP_MACRO
```

**Sources:** [axconfig-macros/tests/example_config.rs(L1 - L87)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-macros/tests/example_config.rs#L1-L87)