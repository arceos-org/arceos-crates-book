# Architecture Overview

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/README.md)
> * [src/lib.rs](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs)

This document provides a high-level view of the `riscv_goldfish` system architecture, showing how the RTC driver components integrate from the hardware layer up to application interfaces. It covers the overall system stack, component relationships, and data flow patterns without diving into implementation details.

For detailed API documentation, see [API Reference](/arceos-org/riscv_goldfish/2.1-api-reference). For hardware register specifics, see [Hardware Interface](/arceos-org/riscv_goldfish/2.2-hardware-interface). For time conversion implementation details, see [Time Conversion](/arceos-org/riscv_goldfish/2.3-time-conversion).

## System Stack Architecture

The `riscv_goldfish` driver operates within a layered architecture that spans from hardware registers to application-level time services:

```mermaid
flowchart TD
subgraph subGraph4["Hardware Platform"]
    GOLDFISH_HW["Goldfish RTC Hardware"]
end
subgraph subGraph3["Hardware Registers"]
    RTC_TIME_LOW_REG["RTC_TIME_LOW (0x00)"]
    RTC_TIME_HIGH_REG["RTC_TIME_HIGH (0x04)"]
end
subgraph subGraph2["Hardware Abstraction"]
    MMIO_READ["read()"]
    MMIO_WRITE["write()"]
    BASE_ADDR["base_address field"]
end
subgraph subGraph1["Driver Layer (src/lib.rs)"]
    RTC_STRUCT["Rtc struct"]
    API_NEW["new()"]
    API_GET["get_unix_timestamp()"]
    API_SET["set_unix_timestamp()"]
end
subgraph subGraph0["Application Layer"]
    APP["Application Code"]
    OS["ArceOS Operating System"]
end

API_GET --> MMIO_READ
API_SET --> MMIO_WRITE
APP --> OS
MMIO_READ --> BASE_ADDR
MMIO_READ --> RTC_TIME_HIGH_REG
MMIO_READ --> RTC_TIME_LOW_REG
MMIO_WRITE --> BASE_ADDR
MMIO_WRITE --> RTC_TIME_HIGH_REG
MMIO_WRITE --> RTC_TIME_LOW_REG
OS --> RTC_STRUCT
RTC_STRUCT --> API_GET
RTC_STRUCT --> API_NEW
RTC_STRUCT --> API_SET
RTC_TIME_HIGH_REG --> GOLDFISH_HW
RTC_TIME_LOW_REG --> GOLDFISH_HW
```

This architecture demonstrates the clear separation of concerns between application logic, driver abstraction, hardware interface, and physical hardware. The `Rtc` struct serves as the primary abstraction boundary, providing a safe interface to unsafe hardware operations.

**Sources: [src/lib.rs(L11 - L14)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L11-L14) [src/lib.rs(L26 - L33)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L26-L33) [src/lib.rs(L35 - L49)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L35-L49)**

## Component Architecture

The core driver architecture centers around the `Rtc` struct and its associated methods:

```mermaid
flowchart TD
subgraph subGraph3["Hardware Interface"]
    READ_VOLATILE["core::ptr::read_volatile"]
    WRITE_VOLATILE["core::ptr::write_volatile"]
end
subgraph Constants["Constants"]
    RTC_TIME_LOW_CONST["RTC_TIME_LOW = 0x00"]
    RTC_TIME_HIGH_CONST["RTC_TIME_HIGH = 0x04"]
    NSEC_PER_SEC_CONST["NSEC_PER_SEC = 1_000_000_000"]
end
subgraph subGraph1["Private Implementation"]
    READ_METHOD["read(reg: usize) -> u32"]
    WRITE_METHOD["write(reg: usize, value: u32)"]
    BASE_FIELD["base_address: usize"]
end
subgraph subGraph0["Public API (src/lib.rs)"]
    RTC["Rtc"]
    NEW["new(base_address: usize)"]
    GET_TS["get_unix_timestamp() -> u64"]
    SET_TS["set_unix_timestamp(unix_time: u64)"]
end

GET_TS --> NSEC_PER_SEC_CONST
GET_TS --> READ_METHOD
NEW --> BASE_FIELD
READ_METHOD --> BASE_FIELD
READ_METHOD --> READ_VOLATILE
READ_METHOD --> RTC_TIME_HIGH_CONST
READ_METHOD --> RTC_TIME_LOW_CONST
SET_TS --> NSEC_PER_SEC_CONST
SET_TS --> WRITE_METHOD
WRITE_METHOD --> BASE_FIELD
WRITE_METHOD --> RTC_TIME_HIGH_CONST
WRITE_METHOD --> RTC_TIME_LOW_CONST
WRITE_METHOD --> WRITE_VOLATILE
```

The component design follows a minimal surface area principle with only three public methods exposed. The private `read` and `write` methods encapsulate all unsafe hardware interactions, while constants define the register layout and time conversion factors.

**Sources: [src/lib.rs(L6 - L9)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L6-L9) [src/lib.rs(L12 - L14)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L12-L14) [src/lib.rs(L17 - L23)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L17-L23) [src/lib.rs(L27 - L49)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L27-L49)**

## Data Flow Architecture

The driver implements bidirectional data flow between Unix timestamps and hardware nanosecond values:

```mermaid
flowchart TD
subgraph subGraph2["Hardware Layer"]
    HW_REGISTERS["64-bit nanosecond counter"]
end
subgraph subGraph1["Write Path (set_unix_timestamp)"]
    WRITE_START["set_unix_timestamp(unix_time)"]
    CONVERT_TO_NS["unix_time * NSEC_PER_SEC"]
    SPLIT_HIGH["(time_nanos >> 32) as u32"]
    SPLIT_LOW["time_nanos as u32"]
    WRITE_HIGH["write(RTC_TIME_HIGH, high)"]
    WRITE_LOW["write(RTC_TIME_LOW, low)"]
end
subgraph subGraph0["Read Path (get_unix_timestamp)"]
    READ_START["get_unix_timestamp()"]
    READ_LOW["read(RTC_TIME_LOW) -> u32"]
    READ_HIGH["read(RTC_TIME_HIGH) -> u32"]
    COMBINE["(high << 32) | low"]
    CONVERT_TO_SEC["result / NSEC_PER_SEC"]
    READ_RESULT["return u64"]
end

COMBINE --> CONVERT_TO_SEC
CONVERT_TO_NS --> SPLIT_HIGH
CONVERT_TO_NS --> SPLIT_LOW
CONVERT_TO_SEC --> READ_RESULT
READ_HIGH --> COMBINE
READ_HIGH --> HW_REGISTERS
READ_LOW --> COMBINE
READ_LOW --> HW_REGISTERS
READ_START --> READ_HIGH
READ_START --> READ_LOW
SPLIT_HIGH --> WRITE_HIGH
SPLIT_LOW --> WRITE_LOW
WRITE_HIGH --> HW_REGISTERS
WRITE_LOW --> HW_REGISTERS
WRITE_START --> CONVERT_TO_NS
```

The data flow demonstrates the critical conversion between user-space Unix timestamps (seconds) and hardware nanosecond representation. The 64-bit hardware value must be split across two 32-bit registers for write operations and reconstructed for read operations.

**Sources: [src/lib.rs(L36 - L40)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L36-L40) [src/lib.rs(L43 - L49)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L43-L49) [src/lib.rs(L9)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L9-L9)**

## Integration Architecture

The driver integrates within the broader ArceOS ecosystem and cross-platform build system:

```mermaid
flowchart TD
subgraph subGraph3["Driver Core"]
    RTC_DRIVER["riscv_goldfish::Rtc"]
end
subgraph subGraph2["Platform Integration"]
    DEVICE_TREE["Device Tree (base_addr: 0x101000)"]
    ARCEOS_OS["ArceOS"]
    COMPATIBLE["google,goldfish-rtc"]
end
subgraph subGraph1["Crate Features"]
    NO_STD["#![no_std]"]
    CARGO_TOML["Cargo.toml metadata"]
end
subgraph subGraph0["Build Targets"]
    X86_64_LINUX["x86_64-unknown-linux-gnu"]
    X86_64_NONE["x86_64-unknown-none"]
    RISCV64["riscv64gc-unknown-none-elf"]
    AARCH64["aarch64-unknown-none-softfloat"]
end

ARCEOS_OS --> RTC_DRIVER
CARGO_TOML --> NO_STD
COMPATIBLE --> RTC_DRIVER
DEVICE_TREE --> RTC_DRIVER
NO_STD --> AARCH64
NO_STD --> RISCV64
NO_STD --> X86_64_LINUX
NO_STD --> X86_64_NONE
RTC_DRIVER --> AARCH64
RTC_DRIVER --> RISCV64
RTC_DRIVER --> X86_64_LINUX
RTC_DRIVER --> X86_64_NONE
```

The integration architecture shows how the `no_std` design enables cross-compilation to multiple targets while maintaining compatibility with the ArceOS operating system. Device tree integration provides the necessary `base_address` configuration for hardware discovery.

**Sources: [src/lib.rs(L4)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/src/lib.rs#L4-L4) [README.md(L15 - L32)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/README.md#L15-L32) [README.md(L5)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/README.md#L5-L5) [README.md(L10 - L13)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/README.md#L10-L13)**