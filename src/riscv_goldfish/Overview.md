# Overview

> **Relevant source files**
> * [Cargo.toml](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/Cargo.toml)
> * [README.md](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/README.md)

This document provides comprehensive documentation for the `riscv_goldfish` repository, a specialized Real Time Clock (RTC) driver crate designed for RISC-V systems running on the Goldfish platform. The repository implements a `no_std` compatible driver that provides Unix timestamp functionality through memory-mapped I/O operations.

The `riscv_goldfish` crate serves as a hardware abstraction layer between operating system components and Goldfish RTC hardware, enabling time management capabilities in embedded and bare-metal RISC-V environments. This driver is specifically designed for integration with the ArceOS operating system ecosystem but maintains compatibility across multiple target architectures.

For detailed API documentation and usage examples, see [API Reference](/arceos-org/riscv_goldfish/2.1-api-reference). For information about cross-platform compilation and target support, see [Target Platforms and Cross-Compilation](/arceos-org/riscv_goldfish/3.1-target-platforms-and-cross-compilation).

## Repository Purpose and Scope

The `riscv_goldfish` crate provides a minimal, efficient RTC driver implementation with the following core responsibilities:

|Component|Purpose|Code Entity|
| --- | --- | --- |
|RTC Driver Core|Hardware abstraction and timestamp management|Rtcstruct|
|Memory Interface|Direct hardware register access|MMIO operations|
|Time Conversion|Unix timestamp to nanosecond conversion|get_unix_timestamp(),set_unix_timestamp()|
|Platform Integration|Device tree compatibility|Base address configuration|

The driver operates in a `no_std` environment, making it suitable for embedded systems, kernel-level components, and bare-metal applications across multiple architectures including RISC-V, x86_64, and ARM64.

Sources: [Cargo.toml(L1 - L15)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/Cargo.toml#L1-L15) [README.md(L1 - L33)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/README.md#L1-L33)

## System Architecture

**System Integration Architecture**

```mermaid
flowchart TD
subgraph subGraph3["Hardware Layer"]
    GOLDFISH_RTC["Goldfish RTC Hardware"]
    RTC_REGS["RTC Registers"]
end
subgraph subGraph2["Hardware Abstraction"]
    MMIO["Memory-Mapped I/O"]
    DEVICE_TREE["Device Tree Configuration"]
    BASE_ADDR["base_addr: 0x101000"]
end
subgraph subGraph1["Driver Layer"]
    RTC_CRATE["riscv_goldfish crate"]
    RTC_STRUCT["Rtc struct"]
    API_NEW["Rtc::new(base_addr)"]
    API_GET["get_unix_timestamp()"]
    API_SET["set_unix_timestamp()"]
end
subgraph subGraph0["Application Layer"]
    APP["User Applications"]
    ARCEOS["ArceOS Operating System"]
end

API_GET --> MMIO
API_NEW --> MMIO
API_SET --> MMIO
APP --> ARCEOS
ARCEOS --> RTC_CRATE
BASE_ADDR --> API_NEW
DEVICE_TREE --> BASE_ADDR
MMIO --> RTC_REGS
RTC_CRATE --> RTC_STRUCT
RTC_REGS --> GOLDFISH_RTC
RTC_STRUCT --> API_GET
RTC_STRUCT --> API_NEW
RTC_STRUCT --> API_SET
```

This architecture demonstrates the layered approach from applications down to hardware, with the `Rtc` struct serving as the primary interface point. The `base_addr` parameter from device tree configuration initializes the driver through the `Rtc::new()` constructor.

Sources: [README.md(L10 - L12)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/README.md#L10-L12) [README.md(L24 - L28)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/README.md#L24-L28) [Cargo.toml(L6)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/Cargo.toml#L6-L6)

## Core Components and Data Flow

**RTC Driver Component Mapping**

```mermaid
flowchart TD
subgraph subGraph4["Data Types"]
    UNIX_TIME["Unix timestamp (u64)"]
    NANOSECONDS["Nanoseconds (u64)"]
    NSEC_PER_SEC["NSEC_PER_SEC constant"]
end
subgraph subGraph3["Hardware Interface"]
    MMIO_OPS["MMIO Operations"]
    RTC_TIME_LOW["RTC_TIME_LOW register"]
    RTC_TIME_HIGH["RTC_TIME_HIGH register"]
end
subgraph subGraph2["Public API Methods"]
    NEW_METHOD["new(base_addr: usize)"]
    GET_TIMESTAMP["get_unix_timestamp()"]
    SET_TIMESTAMP["set_unix_timestamp()"]
end
subgraph subGraph1["Core Structures"]
    RTC_STRUCT["Rtc struct"]
    BASE_ADDR_FIELD["base_addr field"]
end
subgraph subGraph0["Source Files"]
    LIB_RS["src/lib.rs"]
    CARGO_TOML["Cargo.toml"]
end

CARGO_TOML --> LIB_RS
GET_TIMESTAMP --> MMIO_OPS
GET_TIMESTAMP --> UNIX_TIME
LIB_RS --> RTC_STRUCT
MMIO_OPS --> RTC_TIME_HIGH
MMIO_OPS --> RTC_TIME_LOW
NANOSECONDS --> NSEC_PER_SEC
NEW_METHOD --> BASE_ADDR_FIELD
RTC_STRUCT --> BASE_ADDR_FIELD
RTC_STRUCT --> GET_TIMESTAMP
RTC_STRUCT --> NEW_METHOD
RTC_STRUCT --> SET_TIMESTAMP
SET_TIMESTAMP --> MMIO_OPS
SET_TIMESTAMP --> UNIX_TIME
UNIX_TIME --> NANOSECONDS
```

The driver implements a clean separation between the public API surface and hardware-specific operations. The `Rtc` struct encapsulates the base address and provides methods that handle the conversion between Unix timestamps and hardware nanosecond representations through direct register manipulation.

Sources: [README.md(L10 - L12)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/README.md#L10-L12) [Cargo.toml(L2)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/Cargo.toml#L2-L2)

## Target Platform Support

The crate supports multiple target architectures through its `no_std` design:

|Target Architecture|Purpose|Compatibility|
| --- | --- | --- |
|riscv64gc-unknown-none-elf|Primary RISC-V target|Bare metal, embedded|
|x86_64-unknown-linux-gnu|Development and testing|Linux userspace|
|x86_64-unknown-none|Bare metal x86_64|Kernel, bootloader|
|aarch64-unknown-none-softfloat|ARM64 embedded|Bare metal ARM|

The driver maintains cross-platform compatibility while providing hardware-specific optimizations for the Goldfish RTC implementation. The `no-std` category classification enables deployment in resource-constrained environments typical of embedded systems.

Sources: [Cargo.toml(L12)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/Cargo.toml#L12-L12) [Cargo.toml(L6)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/Cargo.toml#L6-L6)

## Integration with ArceOS Ecosystem

The `riscv_goldfish` crate is designed as a component within the broader ArceOS operating system project. The driver provides essential time management capabilities required by kernel-level services and system calls. The crate's licensing scheme supports integration with both open-source and commercial projects through its triple license approach (GPL-3.0, Apache-2.0, MulanPSL-2.0).

Device tree integration follows standard practices, with the driver expecting a compatible string of `"google,goldfish-rtc"` and memory-mapped register access at the specified base address. This standardized approach ensures compatibility across different RISC-V platform implementations that include Goldfish RTC hardware.

Sources: [Cargo.toml(L7 - L8)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/Cargo.toml#L7-L8) [Cargo.toml(L11)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/Cargo.toml#L11-L11) [README.md(L24 - L28)&emsp;](https://github.com/arceos-org/riscv_goldfish/blob/61e0493d/README.md#L24-L28)