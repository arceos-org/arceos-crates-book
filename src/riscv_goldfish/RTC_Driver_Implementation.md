# RTC Driver Implementation

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/README.md)
> * [src/lib.rs](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs)

This document covers the core implementation details of the RISC-V Goldfish RTC driver, including the main `Rtc` struct, its methods, and the underlying hardware abstraction mechanisms. The implementation provides a clean interface for reading and writing Unix timestamps to Goldfish RTC hardware through memory-mapped I/O operations.

For complete API documentation and usage examples, see [API Reference](/arceos-org/riscv_goldfish/2.1-api-reference). For detailed hardware register mapping and MMIO specifics, see [Hardware Interface](/arceos-org/riscv_goldfish/2.2-hardware-interface). For in-depth time conversion algorithms, see [Time Conversion](/arceos-org/riscv_goldfish/2.3-time-conversion).

## Core Driver Structure

The RTC driver is implemented as a single `Rtc` struct that encapsulates all functionality for interfacing with the Goldfish RTC hardware. The driver follows a minimalist design pattern suitable for embedded and bare-metal environments.

```mermaid
flowchart TD
subgraph subGraph2["Hardware Constants"]
    RtcTimeLow["RTC_TIME_LOW = 0x00"]
    RtcTimeHigh["RTC_TIME_HIGH = 0x04"]
    NsecPerSec["NSEC_PER_SEC = 1_000_000_000"]
end
subgraph subGraph1["Internal Methods"]
    ReadMethod["read(reg: usize) -> u32"]
    WriteMethod["write(reg: usize, value: u32)"]
end
subgraph subGraph0["Rtc Struct Implementation"]
    RtcStruct["Rtc { base_address: usize }"]
    NewMethod["new(base_address: usize) -> Self"]
    GetMethod["get_unix_timestamp() -> u64"]
    SetMethod["set_unix_timestamp(unix_time: u64)"]
end

GetMethod --> NsecPerSec
GetMethod --> ReadMethod
NewMethod --> RtcStruct
ReadMethod --> RtcTimeHigh
ReadMethod --> RtcTimeLow
SetMethod --> NsecPerSec
SetMethod --> WriteMethod
WriteMethod --> RtcTimeHigh
WriteMethod --> RtcTimeLow
```

**Sources:** [src/lib.rs(L11 - L50)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L11-L50)

## Memory-Mapped I/O Layer

The driver implements a low-level hardware abstraction through unsafe memory operations. All hardware access is channeled through two core methods that handle volatile memory operations to prevent compiler optimizations from interfering with hardware communication.

|Method|Purpose|Safety|Return Type|
| --- | --- | --- | --- |
|read(reg: usize)|Read 32-bit value from hardware register|Unsafe - requires valid base address|u32|
|write(reg: usize, value: u32)|Write 32-bit value to hardware register|Unsafe - requires valid base address|()|

```mermaid
flowchart TD
subgraph subGraph3["Physical Hardware"]
    RtcHardware["Goldfish RTC Registers"]
end
subgraph subGraph2["Hardware Layer"]
    VolatileRead["core::ptr::read_volatile()"]
    VolatileWrite["core::ptr::write_volatile()"]
    BaseAddr["(base_address + reg) as *const/*mut u32"]
end
subgraph subGraph1["Abstraction Layer"]
    ReadReg["read(reg)"]
    WriteReg["write(reg, value)"]
end
subgraph subGraph0["Software Layer"]
    GetTimestamp["get_unix_timestamp()"]
    SetTimestamp["set_unix_timestamp()"]
end

BaseAddr --> RtcHardware
GetTimestamp --> ReadReg
ReadReg --> VolatileRead
SetTimestamp --> WriteReg
VolatileRead --> BaseAddr
VolatileWrite --> BaseAddr
WriteReg --> VolatileWrite
```

**Sources:** [src/lib.rs(L17 - L24)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L17-L24)

## Driver Initialization and Configuration

The `Rtc::new()` constructor provides the single entry point for driver instantiation. The method requires a base memory address that corresponds to the hardware device's memory-mapped location, typically obtained from device tree parsing during system initialization.

```mermaid
flowchart TD
subgraph subGraph1["Usage Pattern"]
    CreateInstance["let rtc = Rtc::new(0x101000)"]
    GetTime["rtc.get_unix_timestamp()"]
    SetTime["rtc.set_unix_timestamp(epoch)"]
end
subgraph subGraph0["Initialization Flow"]
    DeviceTree["Device Treertc@101000"]
    BaseAddr["base_addr = 0x101000"]
    NewCall["Rtc::new(base_addr)"]
    RtcInstance["Rtc { base_address: 0x101000 }"]
end

BaseAddr --> NewCall
CreateInstance --> GetTime
CreateInstance --> SetTime
DeviceTree --> BaseAddr
NewCall --> RtcInstance
RtcInstance --> CreateInstance
```

The constructor performs no hardware validation or initialization - it simply stores the provided base address for future memory operations. This design assumes that hardware initialization and memory mapping have been handled by the system's device management layer.

**Sources:** [src/lib.rs(L27 - L33)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L27-L33) [README.md(L10 - L16)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/README.md#L10-L16)

## 64-bit Register Handling

The Goldfish RTC hardware stores time as a 64-bit nanosecond value split across two consecutive 32-bit registers. The driver handles this split through careful register sequencing and bit manipulation to maintain data integrity during multi-register operations.

```mermaid
flowchart TD
subgraph subGraph1["64-bit Write Operation"]
    subgraph subGraph0["64-bit Read Operation"]
        InputUnix["unix_time: u64"]
        ConvertNanos["unix_time * NSEC_PER_SEC"]
        SplitHigh["(time_nanos >> 32) as u32"]
        SplitLow["time_nanos as u32"]
        WriteHigh["write(RTC_TIME_HIGH, high)"]
        WriteLow["write(RTC_TIME_LOW, low)"]
        ReadLow["read(RTC_TIME_LOW) -> u32"]
        CombineBits["(high << 32) | low"]
        ReadHigh["read(RTC_TIME_HIGH) -> u32"]
        ConvertTime["nanoseconds / NSEC_PER_SEC"]
        ReturnUnix["return unix_timestamp"]
    end
end

CombineBits --> ConvertTime
ConvertNanos --> SplitHigh
ConvertNanos --> SplitLow
ConvertTime --> ReturnUnix
InputUnix --> ConvertNanos
ReadHigh --> CombineBits
ReadLow --> CombineBits
SplitHigh --> WriteHigh
SplitLow --> WriteLow
```

The write operation follows a specific sequence where the high register is written first, followed by the low register. This ordering ensures atomic updates from the hardware perspective and prevents temporal inconsistencies during timestamp updates.

**Sources:** [src/lib.rs(L36 - L49)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L36-L49)

## Error Handling and Safety Model

The driver implementation uses Rust's `unsafe` blocks to perform direct memory access while maintaining memory safety through controlled access patterns. The safety model relies on several assumptions:

* The provided `base_address` points to valid, mapped memory
* The memory region corresponds to actual Goldfish RTC hardware
* Concurrent access is managed by the calling code
* The hardware registers maintain their expected layout

```mermaid
flowchart TD
subgraph subGraph1["Caller Responsibilities"]
    ValidAddr["Provide valid base_address"]
    MemoryMap["Ensure memory mapping"]
    Concurrency["Handle concurrent access"]
end
subgraph subGraph0["Safety Boundaries"]
    SafeAPI["Safe Public API"]
    UnsafeInternal["Unsafe Internal Methods"]
    HardwareLayer["Hardware Memory"]
end

Concurrency --> SafeAPI
MemoryMap --> SafeAPI
SafeAPI --> UnsafeInternal
UnsafeInternal --> HardwareLayer
ValidAddr --> SafeAPI
```

The driver does not perform runtime validation of memory addresses or hardware responses, prioritizing performance and simplicity for embedded use cases where system-level guarantees about hardware presence and memory mapping are typically established during boot.

**Sources:** [src/lib.rs(L17 - L24)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L17-L24)