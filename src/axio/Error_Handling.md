# Error Handling

> **Relevant source files**
> * [src/error.rs](https://github.com/arceos-org/axio/blob/a675e6d5/src/error.rs)
> * [src/lib.rs](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs)

This document covers error handling mechanisms within the `axio` crate, explaining how errors are defined, propagated, and handled across I/O operations. The error system provides consistent error reporting for `no_std` environments while maintaining compatibility with standard I/O error patterns.

For information about the core I/O traits that use these error types, see [Core I/O Traits](/arceos-org/axio/2-core-io-traits). For details about specific implementations that handle errors, see [Implementations](/arceos-org/axio/4-implementations).

## Error Type System

The `axio` crate implements a minimalist error facade over the `axerrno` crate, providing consistent error handling across all I/O operations. The error system consists of two primary types that are re-exported from `axerrno`.

### Core Error Types

|Type|Source|Purpose|
| --- | --- | --- |
|Error|axerrno::AxError|Represents all possible I/O error conditions|
|Result<T>|axerrno::AxResult<T>|Standard result type for I/O operations|

```mermaid
flowchart TD
subgraph subGraph2["I/O Trait Methods"]
    READ_METHOD["Read::read()"]
    WRITE_METHOD["Write::write()"]
    SEEK_METHOD["Seek::seek()"]
    FILL_BUF_METHOD["BufRead::fill_buf()"]
end
subgraph subGraph1["axerrno Crate"]
    AX_ERROR["AxError"]
    AX_RESULT["AxResult<T>"]
end
subgraph subGraph0["axio Error System"]
    ERROR_MOD["src/error.rs"]
    ERROR_TYPE["Error"]
    RESULT_TYPE["Result<T>"]
end

ERROR_MOD --> ERROR_TYPE
ERROR_MOD --> RESULT_TYPE
ERROR_TYPE --> AX_ERROR
FILL_BUF_METHOD --> RESULT_TYPE
READ_METHOD --> RESULT_TYPE
RESULT_TYPE --> AX_RESULT
SEEK_METHOD --> RESULT_TYPE
WRITE_METHOD --> RESULT_TYPE
```

Sources: [src/error.rs(L1 - L3)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/error.rs#L1-L3) [src/lib.rs(L19)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L19-L19)

### Error Categories and Usage Patterns

The `axio` crate utilizes specific error variants through the `ax_err!` macro to provide meaningful error information for different failure scenarios.

```mermaid
flowchart TD
subgraph subGraph3["Buffered Operations"]
    READ_UNTIL["read_until()"]
    READ_LINE["read_line()"]
    WOULD_BLOCK["WouldBlock"]
end
subgraph subGraph2["Write Operations"]
    WRITE_ALL["write_all()"]
    WRITE_FMT["write_fmt()"]
    WRITE_ZERO["WriteZero"]
    INVALID_DATA["InvalidData"]
end
subgraph subGraph1["Read Operations"]
    READ_EXACT["read_exact()"]
    READ_TO_END["read_to_end()"]
    UNEXPECTED_EOF["UnexpectedEof"]
end
subgraph subGraph0["Memory Operations"]
    MEM_ALLOC["Memory Allocation"]
    NO_MEMORY["NoMemory"]
end

MEM_ALLOC --> NO_MEMORY
READ_EXACT --> UNEXPECTED_EOF
READ_LINE --> INVALID_DATA
READ_TO_END --> NO_MEMORY
READ_UNTIL --> WOULD_BLOCK
WRITE_ALL --> WRITE_ZERO
WRITE_FMT --> INVALID_DATA
```

Sources: [src/lib.rs(L89)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L89-L89) [src/lib.rs(L183)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L183-L183) [src/lib.rs(L203)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L203-L203) [src/lib.rs(L244)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L244-L244) [src/lib.rs(L328)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L328-L328) [src/lib.rs(L366)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L366-L366)

## Error Creation and Propagation

### Theax_err!Macro Pattern

The `axio` crate consistently uses the `ax_err!` macro from `axerrno` to create structured errors with context information. This macro allows for both simple error creation and error wrapping.

|Error Type|Usage Context|Location|
| --- | --- | --- |
|NoMemory|Vector allocation failure|src/lib.rs89|
|UnexpectedEof|Incomplete buffer fill|src/lib.rs183|
|WriteZero|Failed complete write|src/lib.rs203|
|InvalidData|Format/UTF-8 errors|src/lib.rs244src/lib.rs366|

### Error Flow in I/O Operations

```mermaid
flowchart TD
subgraph subGraph3["Error Propagation"]
    QUESTION_MARK["? operator"]
    EARLY_RETURN["Early return"]
    RESULT_TYPE_RETURN["Result<T> return"]
end
subgraph subGraph2["Error Creation"]
    AX_ERR_MACRO["ax_err! macro"]
    ERROR_CONTEXT["Error with context"]
end
subgraph subGraph1["Error Detection Points"]
    MEM_CHECK["Memory Allocation"]
    BUFFER_CHECK["Buffer State"]
    DATA_CHECK["Data Validation"]
    BLOCKING_CHECK["Non-blocking State"]
end
subgraph subGraph0["Operation Types"]
    READ_OP["Read Operation"]
    WRITE_OP["Write Operation"]
    SEEK_OP["Seek Operation"]
    BUF_OP["Buffered Operation"]
end
IO_OP["I/O Operation Initiated"]

AX_ERR_MACRO --> QUESTION_MARK
BLOCKING_CHECK --> ERROR_CONTEXT
BUFFER_CHECK --> AX_ERR_MACRO
BUF_OP --> BLOCKING_CHECK
BUF_OP --> DATA_CHECK
BUF_OP --> MEM_CHECK
DATA_CHECK --> AX_ERR_MACRO
EARLY_RETURN --> RESULT_TYPE_RETURN
ERROR_CONTEXT --> QUESTION_MARK
IO_OP --> BUF_OP
IO_OP --> READ_OP
IO_OP --> SEEK_OP
IO_OP --> WRITE_OP
MEM_CHECK --> AX_ERR_MACRO
QUESTION_MARK --> EARLY_RETURN
READ_OP --> BUFFER_CHECK
WRITE_OP --> BUFFER_CHECK
```

Sources: [src/lib.rs(L24)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L24-L24) [src/lib.rs(L89)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L89-L89) [src/lib.rs(L183)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L183-L183) [src/lib.rs(L203)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L203-L203) [src/lib.rs(L244)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L244-L244) [src/lib.rs(L328)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L328-L328) [src/lib.rs(L366)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L366-L366)

## Specific Error Handling Implementations

### Memory Management Errors

The `default_read_to_end` function demonstrates sophisticated error handling for memory allocation failures in `no_std` environments.

```mermaid
flowchart TD
subgraph subGraph1["Error Handling Flow"]
    ALLOCATION_FAIL["Allocation Fails"]
    ERROR_WRAPPING["Error Wrapping"]
    EARLY_RETURN["Return Err(Error)"]
end
subgraph subGraph0["default_read_to_end Function"]
    START["Function Entry"]
    TRY_RESERVE["buf.try_reserve(PROBE_SIZE)"]
    CHECK_RESULT["Check Result"]
    WRAP_ERROR["ax_err!(NoMemory, e)"]
    CONTINUE["Continue Operation"]
end

ALLOCATION_FAIL --> WRAP_ERROR
CHECK_RESULT --> ALLOCATION_FAIL
CHECK_RESULT --> CONTINUE
ERROR_WRAPPING --> EARLY_RETURN
START --> TRY_RESERVE
TRY_RESERVE --> CHECK_RESULT
WRAP_ERROR --> ERROR_WRAPPING
```

Sources: [src/lib.rs(L88 - L90)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L88-L90)

### Buffer State Validation

The `read_exact` method implements comprehensive error handling for incomplete read operations.

```mermaid
sequenceDiagram
    participant Client as Client
    participant read_exact as read_exact
    participant read_method as read_method
    participant ax_err_macro as ax_err_macro

    Client ->> read_exact: read_exact(&mut buf)
    loop "While buffer not
    loop empty"
        read_exact ->> read_method: read(buf)
    alt "Read returns 0"
        read_method -->> read_exact: Ok(0)
        read_exact ->> read_exact: break loop
    else "Read returns n > 0"
        read_method -->> read_exact: Ok(n)
        read_exact ->> read_exact: advance buffer
    else "Read returns error"
        read_method -->> read_exact: Err(e)
        read_exact -->> Client: Err(e)
    end
    end
    end
    alt "Buffer still has data"
        read_exact ->> ax_err_macro: ax_err!(UnexpectedEof, message)
        ax_err_macro -->> read_exact: Error
        read_exact -->> Client: Err(Error)
    else "Buffer fully consumed"
        read_exact -->> Client: Ok(())
    end
```

Sources: [src/lib.rs(L171 - L187)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L171-L187)

### Non-blocking I/O Error Handling

The `read_until` method demonstrates special handling for `WouldBlock` errors in non-blocking scenarios.

|Error Type|Handling Strategy|Code Location|
| --- | --- | --- |
|WouldBlock|Continue loop, retry operation|src/lib.rs328|
|Other errors|Propagate immediately|src/lib.rs329|

```mermaid
flowchart TD
subgraph subGraph0["read_until Error Handling"]
    FILL_BUF["fill_buf()"]
    MATCH_RESULT["Match Result"]
    WOULD_BLOCK_CASE["WouldBlock"]
    OTHER_ERROR["Other Error"]
    CONTINUE_LOOP["continue"]
    RETURN_ERROR["return Err(e)"]
end

FILL_BUF --> MATCH_RESULT
MATCH_RESULT --> OTHER_ERROR
MATCH_RESULT --> WOULD_BLOCK_CASE
OTHER_ERROR --> RETURN_ERROR
WOULD_BLOCK_CASE --> CONTINUE_LOOP
```

Sources: [src/lib.rs(L325 - L329)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L325-L329)

## Integration with axerrno

The `axio` error system serves as a thin facade over the `axerrno` crate, providing domain-specific error handling while maintaining the underlying error infrastructure.

### Re-export Pattern

The error module uses a simple re-export pattern to maintain API consistency while delegating actual error functionality to `axerrno`.

```mermaid
flowchart TD
subgraph subGraph2["axerrno Crate Features"]
    ERROR_CODES["Error Code Definitions"]
    ERROR_CONTEXT["Error Context Support"]
    MACRO_SUPPORT["ax_err! Macro"]
end
subgraph subGraph1["src/error.rs Implementation"]
    AXERROR_IMPORT["axerrno::AxError as Error"]
    AXRESULT_IMPORT["axerrno::AxResult as Result"]
end
subgraph subGraph0["axio Public API"]
    PUBLIC_ERROR["pub use Error"]
    PUBLIC_RESULT["pub use Result"]
end

AXERROR_IMPORT --> ERROR_CODES
AXRESULT_IMPORT --> ERROR_CODES
MACRO_SUPPORT --> ERROR_CONTEXT
PUBLIC_ERROR --> AXERROR_IMPORT
PUBLIC_RESULT --> AXRESULT_IMPORT
```

Sources: [src/error.rs(L1 - L2)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/error.rs#L1-L2) [src/lib.rs(L24)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L24-L24)

## Error Handling Best Practices

The `axio` codebase demonstrates several consistent patterns for error handling in `no_std` I/O operations:

1. **Immediate Error Propagation**: Use the `?` operator for most error conditions
2. **Contextual Error Creation**: Use `ax_err!` macro with descriptive messages
3. **Special Case Handling**: Handle `WouldBlock` errors differently from fatal errors
4. **Memory Safety**: Wrap allocation errors with appropriate context
5. **Data Validation**: Validate UTF-8 and format correctness with `InvalidData` errors

These patterns ensure consistent error behavior across all I/O traits while maintaining the lightweight design required for `no_std` environments.

Sources: [src/lib.rs(L24)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L24-L24) [src/lib.rs(L89)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L89-L89) [src/lib.rs(L183)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L183-L183) [src/lib.rs(L203)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L203-L203) [src/lib.rs(L244)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L244-L244) [src/lib.rs(L328)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L328-L328) [src/lib.rs(L366)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L366-L366) [src/error.rs(L1 - L3)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/error.rs#L1-L3)