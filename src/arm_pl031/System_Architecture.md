# System Architecture

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/README.md)
> * [src/lib.rs](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs)

## Purpose and Scope

This document explains the high-level architecture of the arm_pl031 crate, focusing on how the driver abstracts ARM PL031 RTC hardware through layered components, safety boundaries, and feature integration. It covers the structural relationships between the core `Rtc` driver, hardware register interface, and optional feature extensions.

For implementation details of specific components, see [Core Driver Implementation](/arceos-org/arm_pl031/3-core-driver-implementation). For practical usage guidance, see [Getting Started](/arceos-org/arm_pl031/2-getting-started).

## Overall Architecture Overview

The arm_pl031 crate implements a layered architecture that provides safe, high-level access to ARM PL031 RTC hardware while maintaining clear separation between abstraction levels.

```mermaid
flowchart TD
subgraph subGraph3["Safety Boundary"]
    UNSAFE_OPS["unsafe operations"]
end
subgraph subGraph2["Hardware Layer"]
    MMIO["Memory-Mapped I/O"]
    PL031_HW["ARM PL031 Hardware"]
end
subgraph subGraph1["arm_pl031 Crate"]
    RTC["Rtc struct"]
    REGISTERS["Registers struct"]
    CHRONO_MOD["chrono module"]
end
subgraph subGraph0["Application Layer"]
    APP["Application Code"]
    CHRONO_API["chrono::DateTime API"]
    UNIX_API["Unix Timestamp API"]
end

APP --> CHRONO_API
APP --> UNIX_API
CHRONO_API --> CHRONO_MOD
CHRONO_MOD --> RTC
MMIO --> PL031_HW
REGISTERS --> MMIO
REGISTERS --> UNSAFE_OPS
RTC --> REGISTERS
UNIX_API --> RTC
UNSAFE_OPS --> MMIO
```

**Architecture Diagram: Overall System Layers**

This architecture provides multiple abstraction levels, allowing developers to choose between high-level `DateTime` operations or low-level timestamp manipulation while maintaining memory safety through controlled unsafe boundaries.

Sources: [src/lib.rs(L1 - L159)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L1-L159)

## Hardware Abstraction Layers

The driver implements a three-tier abstraction model that isolates hardware specifics while providing flexible access patterns.

```mermaid
flowchart TD
subgraph subGraph2["Hardware Interface"]
    READ_VOLATILE["read_volatile()"]
    WRITE_VOLATILE["write_volatile()"]
    ADDR_OF["addr_of!()"]
    ADDR_OF_MUT["addr_of_mut!()"]
end
subgraph subGraph1["Register Abstraction"]
    DR["dr: u32"]
    LR["lr: u32"]
    MR["mr: u32"]
    IMSC["imsc: u8"]
    MIS["mis: u8"]
    ICR["icr: u8"]
end
subgraph subGraph0["Safe Rust Interface"]
    GET_TIME["get_unix_timestamp()"]
    SET_TIME["set_unix_timestamp()"]
    INTERRUPTS["enable_interrupt()"]
    MATCH_OPS["set_match_timestamp()"]
end

DR --> READ_VOLATILE
GET_TIME --> DR
IMSC --> WRITE_VOLATILE
INTERRUPTS --> IMSC
LR --> WRITE_VOLATILE
MATCH_OPS --> MR
MR --> WRITE_VOLATILE
READ_VOLATILE --> ADDR_OF
SET_TIME --> LR
WRITE_VOLATILE --> ADDR_OF_MUT
```

**Architecture Diagram: Hardware Abstraction Tiers**

Each tier serves a specific purpose:

* **Safe Interface**: Provides memory-safe methods with clear semantics
* **Register Abstraction**: Maps PL031 registers to Rust types with proper layout
* **Hardware Interface**: Handles volatile memory operations with explicit safety documentation

Sources: [src/lib.rs(L15 - L39)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L15-L39) [src/lib.rs(L62 - L121)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L62-L121)

## Component Architecture and Relationships

The core architecture centers around two primary structures that encapsulate hardware interaction and provide the driver interface.

|Component|Purpose|Safety Level|Key Responsibilities|
| --- | --- | --- | --- |
|Rtc|Driver interface|Safe wrapper|Time operations, interrupt management|
|Registers|Hardware layout|Memory representation|Register field mapping, memory alignment|
|chronomodule|Feature extension|Safe wrapper|DateTime conversion, high-level API|

```mermaid
flowchart TD
subgraph subGraph2["Memory Operations"]
    PTR_DEREF["(*self.registers)"]
    VOLATILE_READ["addr_of!(field).read_volatile()"]
    VOLATILE_WRITE["addr_of_mut!(field).write_volatile(value)"]
end
subgraph subGraph1["Register Layout Structure"]
    REG_STRUCT["#[repr(C, align(4))] Registers"]
    DR_FIELD["dr: u32"]
    MR_FIELD["mr: u32"]
    LR_FIELD["lr: u32"]
    CONTROL_FIELDS["cr, imsc, ris, mis, icr: u8"]
    RESERVED["_reserved0..4: [u8; 3]"]
end
subgraph subGraph0["Rtc Driver Structure"]
    RTC_STRUCT["Rtc { registers: *mut Registers }"]
    NEW_METHOD["unsafe fn new(base_address: *mut u32)"]
    GET_TIMESTAMP["fn get_unix_timestamp() -> u32"]
    SET_TIMESTAMP["fn set_unix_timestamp(unix_time: u32)"]
    INTERRUPT_METHODS["interrupt management methods"]
end

GET_TIMESTAMP --> PTR_DEREF
INTERRUPT_METHODS --> PTR_DEREF
NEW_METHOD --> RTC_STRUCT
PTR_DEREF --> VOLATILE_READ
PTR_DEREF --> VOLATILE_WRITE
RTC_STRUCT --> REG_STRUCT
SET_TIMESTAMP --> PTR_DEREF
VOLATILE_READ --> DR_FIELD
VOLATILE_WRITE --> LR_FIELD
VOLATILE_WRITE --> MR_FIELD
```

**Architecture Diagram: Core Component Relationships**

The `Rtc` struct maintains a single pointer to a `Registers` structure, which provides C-compatible memory layout matching the PL031 hardware specification. All hardware access flows through volatile pointer operations to ensure proper memory semantics.

Sources: [src/lib.rs(L15 - L44)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L15-L44) [src/lib.rs(L46 - L61)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L46-L61)

## Safety Model and Boundaries

The driver implements a clear safety model that contains all unsafe operations at the hardware interface boundary while providing safe abstractions for application code.

```mermaid
flowchart TD
subgraph subGraph2["Hardware Zone"]
    MMIO_ACCESS["Direct MMIO register access"]
    DEVICE_MEMORY["PL031 device memory"]
end
subgraph subGraph1["Unsafe Boundary"]
    UNSAFE_NEW["unsafe fn new()"]
    SAFETY_INVARIANTS["Safety invariants and documentation"]
    VOLATILE_OPS["Volatile memory operations"]
end
subgraph subGraph0["Safe Zone"]
    SAFE_METHODS["Public safe methods"]
    TIMESTAMP_OPS["get_unix_timestamp, set_unix_timestamp"]
    INTERRUPT_OPS["enable_interrupt, clear_interrupt"]
    MATCH_OPS["set_match_timestamp, matched"]
end
subgraph subGraph3["Concurrency Safety"]
    SEND_IMPL["unsafe impl Send for Rtc"]
    THREAD_SAFETY["Multi-thread access guarantees"]
    SYNC_IMPL["unsafe impl Sync for Rtc"]
end
UNSAFE_BOUNDARY["UNSAFE_BOUNDARY"]

INTERRUPT_OPS --> VOLATILE_OPS
MATCH_OPS --> VOLATILE_OPS
MMIO_ACCESS --> DEVICE_MEMORY
SAFE_METHODS --> UNSAFE_BOUNDARY
SEND_IMPL --> THREAD_SAFETY
SYNC_IMPL --> THREAD_SAFETY
TIMESTAMP_OPS --> VOLATILE_OPS
UNSAFE_NEW --> SAFETY_INVARIANTS
VOLATILE_OPS --> MMIO_ACCESS
```

**Architecture Diagram: Safety Boundaries and Concurrency Model**

### Safety Requirements

The driver enforces specific safety requirements at the boundary:

1. **Memory Safety**: The `base_address` parameter in `Rtc::new()` must point to valid PL031 MMIO registers
2. **Alignment**: Hardware addresses must be 4-byte aligned as enforced by `#[repr(C, align(4))]`
3. **Exclusive Access**: No other aliases to the device memory region are permitted
4. **Device Mapping**: Memory must be mapped as device memory, not normal cacheable memory

### Concurrency Safety

The driver provides thread-safety through carefully designed `Send` and `Sync` implementations:

* `Send`: Safe because device memory access works from any thread context
* `Sync`: Safe because shared read access to device registers is inherently safe, and mutation requires `&mut self`

Sources: [src/lib.rs(L51 - L60)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L51-L60) [src/lib.rs(L123 - L128)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L123-L128)

## Feature Integration Architecture

The crate supports optional feature integration through Cargo feature flags, with the chrono integration serving as the primary example of the extensibility model.

```

```

**Architecture Diagram: Feature Integration and Build Configuration**

### Feature Design Principles

1. **Minimal Core**: The base driver remains lightweight with only essential functionality
2. **Optional Extensions**: Advanced features like DateTime support are opt-in through feature flags
3. **No-std Compatibility**: All features maintain compatibility with embedded environments
4. **Layered Integration**: Feature modules build upon the core API without modifying its interface

The chrono integration demonstrates this pattern by providing higher-level `DateTime<Utc>` operations while delegating to the core `get_unix_timestamp()` and `set_unix_timestamp()` methods.

Sources: [src/lib.rs(L10 - L11)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L10-L11) [src/lib.rs(L3)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L3-L3)