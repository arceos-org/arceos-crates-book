# Core I/O Traits

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs)

This page documents the four fundamental I/O traits that form the foundation of the `axio` library: `Read`, `Write`, `Seek`, and `BufRead`. These traits provide a `std::io`-compatible interface for performing I/O operations in `no_std` environments.

The traits are designed to mirror Rust's standard library I/O traits while supporting resource-constrained environments through optional allocation-dependent features. For information about concrete implementations of these traits, see [Implementations](/arceos-org/axio/4-implementations). For details about the crate's feature configuration and dependency management, see [Crate Configuration and Features](/arceos-org/axio/3-crate-configuration-and-features).

## Trait Hierarchy and Relationships

The core I/O traits in `axio` form a well-structured hierarchy where `BufRead` extends `Read`, while `Write` and `Seek` operate independently. The following diagram shows the relationships between traits and their key methods:

```mermaid
flowchart TD
Read["Read"]
ReadMethods["read()read_exact()read_to_end()read_to_string()"]
Write["Write"]
WriteMethods["write()flush()write_all()write_fmt()"]
Seek["Seek"]
SeekMethods["seek()rewind()stream_position()"]
BufRead["BufRead"]
BufReadMethods["fill_buf()consume()has_data_left()read_until()read_line()"]
ResultType["Result"]
SeekFrom["SeekFromStart(u64)End(i64)Current(i64)"]
AllocFeature["alloc feature"]
ReadToEnd["read_to_end()read_to_string()read_until()read_line()"]

AllocFeature --> ReadToEnd
BufRead --> BufReadMethods
BufRead --> Read
BufReadMethods --> ResultType
Read --> ReadMethods
ReadMethods --> ResultType
Seek --> SeekMethods
SeekMethods --> ResultType
SeekMethods --> SeekFrom
Write --> WriteMethods
WriteMethods --> ResultType
```

Sources: [src/lib.rs(L152 - L355)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L152-L355)

## The Read Trait

The `Read` trait provides the fundamental interface for reading bytes from a source. It defines both required and optional methods, with some methods only available when the `alloc` feature is enabled.

### Core Methods

|Method|Signature|Description|Feature Requirement|
| --- | --- | --- | --- |
|read|fn read(&mut self, buf: &mut [u8]) -> Result<usize>|Required method to read bytes into buffer|None|
|read_exact|fn read_exact(&mut self, buf: &mut [u8]) -> Result<()>|Read exact number of bytes to fill buffer|None|
|read_to_end|fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize>|Read all bytes until EOF|alloc|
|read_to_string|fn read_to_string(&mut self, buf: &mut String) -> Result<usize>|Read all bytes as UTF-8 string|alloc|

The `read` method is the only required implementation. The `read_exact` method provides a default implementation that repeatedly calls `read` until the buffer is filled or EOF is reached. When EOF is encountered before the buffer is filled, it returns an `UnexpectedEof` error.

```mermaid
flowchart TD
ReadTrait["Read trait"]
RequiredRead["read(&mut self, buf: &mut [u8])"]
ProvidedMethods["Provided implementations"]
ReadExact["read_exact()"]
AllocMethods["Allocation-dependent methods"]
ReadToEnd["read_to_end()"]
ReadToString["read_to_string()"]
ReadExactLoop["Loop calling read() until buffer filled"]
DefaultReadToEnd["default_read_to_end() function"]
AppendToString["append_to_string() helper"]
UnexpectedEof["UnexpectedEof error if incomplete"]
SmallProbe["Small probe reads for efficiency"]
Utf8Validation["UTF-8 validation"]

AllocMethods --> ReadToEnd
AllocMethods --> ReadToString
AppendToString --> Utf8Validation
DefaultReadToEnd --> SmallProbe
ProvidedMethods --> AllocMethods
ProvidedMethods --> ReadExact
ReadExact --> ReadExactLoop
ReadExactLoop --> UnexpectedEof
ReadToEnd --> DefaultReadToEnd
ReadToString --> AppendToString
ReadTrait --> ProvidedMethods
ReadTrait --> RequiredRead
```

Sources: [src/lib.rs(L152 - L188)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L152-L188)

### Memory-Efficient Reading Strategy

The `default_read_to_end` function implements an optimized reading strategy that balances performance with memory usage. It uses several techniques:

* **Small probe reads**: Initial 32-byte reads to avoid unnecessary capacity expansion
* **Adaptive buffer sizing**: Dynamically adjusts read buffer size based on reader behavior
* **Capacity management**: Uses `try_reserve` to handle allocation failures gracefully
* **Short read detection**: Tracks consecutive short reads to optimize buffer sizes

Sources: [src/lib.rs(L26 - L150)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L26-L150)

## The Write Trait

The `Write` trait provides the interface for writing bytes to a destination. Like `Read`, it has one required method with several provided implementations.

### Core Methods

|Method|Signature|Description|
| --- | --- | --- |
|write|fn write(&mut self, buf: &[u8]) -> Result<usize>|Required method to write bytes from buffer|
|flush|fn flush(&mut self) -> Result<()>|Required method to flush buffered data|
|write_all|fn write_all(&mut self, buf: &[u8]) -> Result<()>|Write entire buffer or return error|
|write_fmt|fn write_fmt(&mut self, fmt: fmt::Arguments<'_>) -> Result<()>|Write formatted output|

The `write_all` method ensures that the entire buffer is written by repeatedly calling `write` until completion. If `write` returns 0 (indicating no progress), it returns a `WriteZero` error.

The `write_fmt` method enables formatted output support by implementing a bridge between `fmt::Write` and the I/O `Write` trait through an internal `Adapter` struct.

```mermaid
flowchart TD
WriteTrait["Write trait"]
RequiredMethods["Required methods"]
ProvidedMethods["Provided methods"]
WriteMethod["write()"]
FlushMethod["flush()"]
WriteAll["write_all()"]
WriteFmt["write_fmt()"]
WriteLoop["Loop calling write() until complete"]
WriteZeroError["WriteZero error if no progress"]
AdapterStruct["Adapter struct"]
FmtWriteBridge["Bridge to fmt::Write"]
ErrorTranslation["Translate fmt errors to I/O errors"]

AdapterStruct --> FmtWriteBridge
FmtWriteBridge --> ErrorTranslation
ProvidedMethods --> WriteAll
ProvidedMethods --> WriteFmt
RequiredMethods --> FlushMethod
RequiredMethods --> WriteMethod
WriteAll --> WriteLoop
WriteFmt --> AdapterStruct
WriteLoop --> WriteZeroError
WriteTrait --> ProvidedMethods
WriteTrait --> RequiredMethods
```

Sources: [src/lib.rs(L190 - L249)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L190-L249)

## The Seek Trait

The `Seek` trait provides cursor positioning within streams that support random access. It works with the `SeekFrom` enum to specify different seeking strategies.

### Methods and SeekFrom Enum

|Method|Signature|Description|
| --- | --- | --- |
|seek|fn seek(&mut self, pos: SeekFrom) -> Result<u64>|Required method to seek to specified position|
|rewind|fn rewind(&mut self) -> Result<()>|Convenience method to seek to beginning|
|stream_position|fn stream_position(&mut self) -> Result<u64>|Get current position in stream|

The `SeekFrom` enum defines three positioning strategies:

* `Start(u64)`: Absolute position from beginning
* `End(i64)`: Relative position from end (negative values seek backwards)
* `Current(i64)`: Relative position from current location

```mermaid
flowchart TD
SeekTrait["Seek trait"]
SeekMethod["seek(pos: SeekFrom)"]
ConvenienceMethods["Convenience methods"]
Rewind["rewind()"]
StreamPosition["stream_position()"]
SeekFromEnum["SeekFrom enum"]
StartVariant["Start(u64)"]
EndVariant["End(i64)"]
CurrentVariant["Current(i64)"]
SeekStart["seek(SeekFrom::Start(0))"]
SeekCurrent["seek(SeekFrom::Current(0))"]
PositionResult["Returns new position as u64"]

ConvenienceMethods --> Rewind
ConvenienceMethods --> StreamPosition
Rewind --> SeekStart
SeekFromEnum --> CurrentVariant
SeekFromEnum --> EndVariant
SeekFromEnum --> StartVariant
SeekMethod --> PositionResult
SeekMethod --> SeekFromEnum
SeekTrait --> ConvenienceMethods
SeekTrait --> SeekMethod
StreamPosition --> SeekCurrent
```

Sources: [src/lib.rs(L252 - L301)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L252-L301)

## The BufRead Trait

The `BufRead` trait extends `Read` to provide buffered reading capabilities. It enables efficient line-by-line reading and delimiter-based parsing through its internal buffer management.

### Buffer Management Methods

|Method|Signature|Description|Feature Requirement|
| --- | --- | --- | --- |
|fill_buf|fn fill_buf(&mut self) -> Result<&[u8]>|Required method to access internal buffer|None|
|consume|fn consume(&mut self, amt: usize)|Required method to mark bytes as consumed|None|
|has_data_left|fn has_data_left(&mut self) -> Result<bool>|Check if more data is available|None|
|read_until|fn read_until(&mut self, byte: u8, buf: &mut Vec<u8>) -> Result<usize>|Read until delimiter found|alloc|
|read_line|fn read_line(&mut self, buf: &mut String) -> Result<usize>|Read until newline found|alloc|

The buffered reading pattern follows a fill-consume cycle where `fill_buf` exposes the internal buffer and `consume` marks bytes as processed. This allows for efficient parsing without unnecessary copying.

```mermaid
flowchart TD
BufReadTrait["BufRead trait"]
ExtendsRead["extends Read"]
RequiredBuffer["Required buffer methods"]
ProvidedParsing["Provided parsing methods"]
FillBuf["fill_buf() -> &[u8]"]
Consume["consume(amt: usize)"]
HasDataLeft["has_data_left()"]
AllocParsing["Allocation-dependent parsing"]
ReadUntil["read_until(delimiter)"]
ReadLine["read_line()"]
InternalBuffer["Access to internal buffer"]
MarkProcessed["Mark bytes as processed"]
DelimiterSearch["Search for delimiter byte"]
NewlineSearch["Search for 0xA (newline)"]
FillConsumeLoop["fill_buf/consume loop"]
Utf8Append["UTF-8 string append"]

AllocParsing --> ReadLine
AllocParsing --> ReadUntil
BufReadTrait --> ExtendsRead
BufReadTrait --> ProvidedParsing
BufReadTrait --> RequiredBuffer
Consume --> MarkProcessed
DelimiterSearch --> FillConsumeLoop
FillBuf --> InternalBuffer
NewlineSearch --> Utf8Append
ProvidedParsing --> AllocParsing
ProvidedParsing --> HasDataLeft
ReadLine --> NewlineSearch
ReadUntil --> DelimiterSearch
RequiredBuffer --> Consume
RequiredBuffer --> FillBuf
```

Sources: [src/lib.rs(L303 - L355)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L303-L355)

### Error Handling in Buffered Reading

The `read_until` method demonstrates sophisticated error handling, particularly for the `WouldBlock` error case. When `fill_buf` returns `WouldBlock`, the method continues the loop rather than propagating the error, enabling non-blocking I/O patterns.

Sources: [src/lib.rs(L320 - L347)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L320-L347)

## Supporting Types and Utilities

### PollState Structure

The `PollState` struct provides a simple interface for I/O readiness polling, commonly used in async I/O scenarios:

```css
pub struct PollState {
    pub readable: bool,
    pub writable: bool,
}
```

This structure enables applications to check I/O readiness without blocking, supporting efficient event-driven programming patterns.

### Helper Functions

The crate provides several utility functions that support the trait implementations:

* `default_read_to_end`: Optimized implementation for reading all data with size hints
* `append_to_string`: UTF-8 validation wrapper for string operations

These functions are designed to be reusable across different implementations while maintaining the performance characteristics expected in `no_std` environments.

Sources: [src/lib.rs(L357 - L380)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L357-L380)

## Feature-Gated Functionality

Many methods in the core traits are gated behind the `alloc` feature, which enables dynamic memory allocation. This design allows the library to provide enhanced functionality when memory allocation is available while maintaining core functionality in highly constrained environments.

```mermaid
flowchart TD
CoreTraits["Core trait methods"]
AlwaysAvailable["Always available"]
AllocGated["alloc feature gated"]
ReadBasic["read(), read_exact()"]
WriteBasic["write(), flush(), write_all()"]
SeekAll["All Seek methods"]
BufReadBasic["fill_buf(), consume()"]
ReadDynamic["read_to_end(), read_to_string()"]
BufReadParsing["read_until(), read_line()"]
AllocFeature["alloc feature"]
NoStdBasic["no_std environments"]
Enhanced["Enhanced environments"]

AllocFeature --> AllocGated
AllocGated --> BufReadParsing
AllocGated --> ReadDynamic
AlwaysAvailable --> BufReadBasic
AlwaysAvailable --> ReadBasic
AlwaysAvailable --> SeekAll
AlwaysAvailable --> WriteBasic
CoreTraits --> AllocGated
CoreTraits --> AlwaysAvailable
Enhanced --> AllocGated
Enhanced --> AlwaysAvailable
NoStdBasic --> AlwaysAvailable
```

Sources: [src/lib.rs(L7 - L8)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L7-L8) [src/lib.rs(L21 - L22)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L21-L22) [src/lib.rs(L159 - L168)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L159-L168) [src/lib.rs(L320 - L355)&emsp;](https://github.com/arceos-org/axio/blob/a675e6d5/src/lib.rs#L320-L355)