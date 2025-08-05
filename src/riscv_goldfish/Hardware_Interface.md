# Hardware Interface

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs)

This document covers the low-level hardware interface implementation of the riscv_goldfish RTC driver, focusing on memory-mapped I/O operations, register layout, and the abstraction layer between software and the Goldfish RTC hardware. For higher-level API usage patterns, see [API Reference](/arceos-org/riscv_goldfish/2.1-api-reference). For time conversion logic, see [Time Conversion](/arceos-org/riscv_goldfish/2.3-time-conversion).

## Register Layout and Memory Mapping

The Goldfish RTC hardware exposes a simple memory-mapped interface consisting of two 32-bit registers that together form a 64-bit nanosecond timestamp counter. The driver defines the register offsets as compile-time constants and uses a base address provided during initialization.

### Register Map

|Offset|Register Name|Width|Purpose|
| --- | --- | --- | --- |
|0x00|RTC_TIME_LOW|32-bit|Lower 32 bits of nanosecond timestamp|
|0x04|RTC_TIME_HIGH|32-bit|Upper 32 bits of nanosecond timestamp|

### Memory Address Calculation

```mermaid
flowchart TD
BaseAddr["base_address"]
LowAddr["Low Register Address"]
HighAddr["High Register Address"]
LowReg["RTC_TIME_LOW Register"]
HighReg["RTC_TIME_HIGH Register"]
NS64["64-bit Nanosecond Value"]

BaseAddr --> HighAddr
BaseAddr --> LowAddr
HighAddr --> HighReg
HighReg --> NS64
LowAddr --> LowReg
LowReg --> NS64
```

The driver calculates absolute register addresses by adding the register offset constants to the base address provided during `Rtc::new()` construction.

Sources: [src/lib.rs(L6 - L7)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L6-L7) [src/lib.rs(L31 - L33)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L31-L33)

## Memory-Mapped I/O Operations

The hardware interface implements volatile memory operations to ensure proper communication with the RTC hardware. The driver provides two core memory access functions that handle the low-level register reads and writes.

### Volatile Memory Access Functions

```mermaid
flowchart TD
ReadOp["read(reg: usize)"]
AddrCalc["Address Calculation"]
WriteOp["write(reg: usize, value: u32)"]
ReadPtr["Read Pointer Cast"]
WritePtr["Write Pointer Cast"]
HardwareRead["Hardware Register Read"]
HardwareWrite["Hardware Register Write"]
ReadResult["Return Value"]
WriteResult["Write Complete"]

AddrCalc --> ReadPtr
AddrCalc --> WritePtr
HardwareRead --> ReadResult
HardwareWrite --> WriteResult
ReadOp --> AddrCalc
ReadPtr --> HardwareRead
WriteOp --> AddrCalc
WritePtr --> HardwareWrite
```

The `read()` function performs volatile reads to prevent compiler optimizations that might cache register values, while `write()` ensures immediate hardware updates. Both operations are marked `unsafe` due to raw pointer manipulation.

Sources: [src/lib.rs(L17 - L23)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L17-L23)

## 64-bit Value Handling Across 32-bit Registers

The Goldfish RTC hardware stores nanosecond timestamps as a 64-bit value, but exposes this through two separate 32-bit registers. The driver implements bidirectional conversion between the unified 64-bit representation and the split register format.

### Read Operation Data Flow

```mermaid
flowchart TD
subgraph subGraph2["Value Reconstruction"]
    Combine["(high << 32) | low"]
    UnixTime["Unix Timestamp"]
end
subgraph subGraph1["Driver Read Operations"]
    ReadLow["read(RTC_TIME_LOW)"]
    LowU64["low: u64"]
    ReadHigh["read(RTC_TIME_HIGH)"]
    HighU64["high: u64"]
end
subgraph subGraph0["Hardware Registers"]
    LowHW["RTC_TIME_LOW32-bit hardware"]
    HighHW["RTC_TIME_HIGH32-bit hardware"]
end

Combine --> UnixTime
HighHW --> ReadHigh
HighU64 --> Combine
LowHW --> ReadLow
LowU64 --> Combine
ReadHigh --> HighU64
ReadLow --> LowU64
```

### Write Operation Data Flow

```mermaid
flowchart TD
subgraph subGraph3["Hardware Registers"]
    HighHWOut["RTC_TIME_HIGHupdated"]
    LowHWOut["RTC_TIME_LOWupdated"]
end
subgraph subGraph2["Hardware Write"]
    WriteHigh["write(RTC_TIME_HIGH, high)"]
    WriteLow["write(RTC_TIME_LOW, low)"]
end
subgraph subGraph1["Value Splitting"]
    HighPart["High 32 bits"]
    LowPart["Low 32 bits"]
end
subgraph subGraph0["Input Processing"]
    UnixIn["unix_time: u64"]
    Nanos["time_nanos: u64"]
end

HighPart --> WriteHigh
LowPart --> WriteLow
Nanos --> HighPart
Nanos --> LowPart
UnixIn --> Nanos
WriteHigh --> HighHWOut
WriteLow --> LowHWOut
```

The write operation updates the high register first, then the low register, to maintain temporal consistency during the split-register update sequence.

Sources: [src/lib.rs(L36 - L40)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L36-L40) [src/lib.rs(L43 - L49)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L43-L49)

## Hardware Abstraction Layer

The `Rtc` struct provides a clean abstraction over the raw hardware interface, encapsulating the base address and providing safe access patterns for register operations.

### Driver Structure and Hardware Mapping

```mermaid
flowchart TD
subgraph subGraph2["Physical Hardware"]
    GoldfishRTC["Goldfish RTC Device64-bit nanosecond counter"]
end
subgraph subGraph1["Hardware Abstraction"]
    BaseAddr["base_address field"]
    MemoryMap["Memory Mapping"]
    RegOffsets["RTC_TIME_LOWRTC_TIME_HIGH"]
end
subgraph subGraph0["Software Layer"]
    RtcStruct["Rtc { base_address: usize }"]
    PublicAPI["get_unix_timestamp()set_unix_timestamp()"]
    PrivateOps["read(reg)write(reg, value)"]
end

BaseAddr --> MemoryMap
MemoryMap --> GoldfishRTC
PrivateOps --> MemoryMap
PublicAPI --> PrivateOps
RegOffsets --> MemoryMap
RtcStruct --> PublicAPI
```

The abstraction separates hardware-specific details (register offsets, volatile operations) from the public API, allowing safe high-level operations while maintaining direct hardware access efficiency.

Sources: [src/lib.rs(L11 - L14)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L11-L14) [src/lib.rs(L26 - L50)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L26-L50)

## Safety Considerations

The hardware interface implements several safety measures to ensure reliable operation:

* **Volatile Operations**: All register access uses `read_volatile()` and `write_volatile()` to prevent compiler optimizations that could interfere with hardware communication
* **Unsafe Boundaries**: Raw pointer operations are confined to private methods, exposing only safe public APIs
* **Type Safety**: Register values are properly cast between `u32` hardware format and `u64` application format
* **Address Validation**: Base address is stored during construction and used consistently for all register calculations

The driver assumes the provided base address corresponds to valid, mapped Goldfish RTC hardware, typically obtained from device tree parsing during system initialization.

Sources: [src/lib.rs(L17 - L23)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L17-L23) [src/lib.rs(L31 - L33)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L31-L33)