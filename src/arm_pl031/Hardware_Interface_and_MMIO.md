# Hardware Interface and MMIO

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs)

This document details how the ARM PL031 RTC driver interfaces with the physical hardware through Memory-Mapped I/O (MMIO) operations. It covers the register layout, volatile access patterns, memory safety boundaries, and the hardware abstraction layer implementation.

For information about specific register operations and their meanings, see [Register Operations](/arceos-org/arm_pl031/3.3-register-operations). For details about the overall driver architecture, see [Driver Architecture and Design](/arceos-org/arm_pl031/3.1-driver-architecture-and-design).

## MMIO Register Layout

The PL031 hardware interface is accessed through a well-defined memory-mapped register layout represented by the `Registers` struct. This struct provides a direct mapping to the hardware register space with proper alignment and padding.

**PL031 Register Memory Layout**

```mermaid
flowchart TD
subgraph subGraph0["MMIO Address Space"]
    BASE["Base Address(4-byte aligned)"]
    DR["DR (0x00)Data Register32-bit"]
    MR["MR (0x04)Match Register32-bit"]
    LR["LR (0x08)Load Register32-bit"]
    CR["CR (0x0C)Control Register8-bit + 3 reserved"]
    IMSC["IMSC (0x10)Interrupt Mask8-bit + 3 reserved"]
    RIS["RIS (0x14)Raw Interrupt Status8-bit + 3 reserved"]
    MIS["MIS (0x18)Masked Interrupt Status8-bit + 3 reserved"]
    ICR["ICR (0x1C)Interrupt Clear8-bit + 3 reserved"]
end

BASE --> DR
CR --> IMSC
DR --> MR
IMSC --> RIS
LR --> CR
MIS --> ICR
MR --> LR
RIS --> MIS
```

The `Registers` struct enforces proper memory layout through careful use of padding and alignment directives. Each register is positioned according to the PL031 specification, with reserved bytes maintaining proper spacing between hardware registers.

Sources: [src/lib.rs(L15 - L39)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L15-L39)

## Hardware Access Abstraction

The driver implements a three-layer abstraction for hardware access, ensuring both safety and performance while maintaining direct hardware control.

**Hardware Access Layers**

```mermaid
flowchart TD
subgraph subGraph1["Hardware Layer"]
    MMIO["Memory-Mapped I/ODevice Memory Region"]
    REGS["PL031 Hardware RegistersPhysical Hardware"]
end
subgraph subGraph0["Software Layers"]
    API["Public API Methodsget_unix_timestamp()set_unix_timestamp()enable_interrupt()"]
    UNSAFE["Unsafe Hardware Accessaddr_of!()read_volatile()write_volatile()"]
    PTR["Raw Pointer Operations(*self.registers).dr(*self.registers).mr"]
end

API --> UNSAFE
MMIO --> REGS
PTR --> MMIO
UNSAFE --> PTR
```

The `Rtc` struct maintains a single `*mut Registers` pointer that serves as the bridge between safe Rust code and raw hardware access. All hardware operations go through volatile memory operations to ensure proper interaction with device memory.

Sources: [src/lib.rs(L42 - L44)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L42-L44) [src/lib.rs(L56 - L60)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L56-L60)

## Volatile Memory Operations

All hardware register access uses volatile operations to prevent compiler optimizations that could interfere with hardware behavior. The driver employs a consistent pattern for both read and write operations.

**MMIO Operation Patterns**

```mermaid
sequenceDiagram
    participant PublicAPI as "Public API"
    participant UnsafeBlock as "Unsafe Block"
    participant addr_ofaddr_of_mut as "addr_of!/addr_of_mut!"
    participant read_volatilewrite_volatile as "read_volatile/write_volatile"
    participant PL031Hardware as "PL031 Hardware"

    Note over PublicAPI,PL031Hardware: Read Operation Pattern
    PublicAPI ->> UnsafeBlock: Call hardware access
    UnsafeBlock ->> addr_ofaddr_of_mut: addr_of!((*registers).field)
    addr_ofaddr_of_mut ->> read_volatilewrite_volatile: .read_volatile()
    read_volatilewrite_volatile ->> PL031Hardware: Load from device memory
    PL031Hardware -->> read_volatilewrite_volatile: Register value
    read_volatilewrite_volatile -->> addr_ofaddr_of_mut: Raw value
    addr_ofaddr_of_mut -->> UnsafeBlock: Typed value
    UnsafeBlock -->> PublicAPI: Safe result
    Note over PublicAPI,PL031Hardware: Write Operation Pattern
    PublicAPI ->> UnsafeBlock: Call hardware access
    UnsafeBlock ->> addr_ofaddr_of_mut: addr_of_mut!((*registers).field)
    addr_ofaddr_of_mut ->> read_volatilewrite_volatile: .write_volatile(value)
    read_volatilewrite_volatile ->> PL031Hardware: Store to device memory
    PL031Hardware -->> read_volatilewrite_volatile: Write acknowledgment
    read_volatilewrite_volatile -->> addr_ofaddr_of_mut: Write complete
    addr_ofaddr_of_mut -->> UnsafeBlock: Operation complete
    UnsafeBlock -->> PublicAPI: Safe completion
```

The driver uses `addr_of!` and `addr_of_mut!` macros to safely obtain field addresses from the raw pointer without creating intermediate references, avoiding undefined behavior while maintaining direct hardware access.

Sources: [src/lib.rs(L63 - L67)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L63-L67) [src/lib.rs(L70 - L74)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L70-L74) [src/lib.rs(L78 - L82)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L78-L82)

## Memory Safety Model

The hardware interface implements a carefully designed safety model that isolates unsafe operations while providing safe public APIs. The safety boundaries are clearly defined and documented.

|Operation Type|Safety Level|Access Pattern|Usage|
| --- | --- | --- | --- |
|Construction|Unsafe|Rtc::new()|Requires valid MMIO base address|
|Register Read|Safe|get_unix_timestamp()|Volatile read through safe wrapper|
|Register Write|Safe|set_unix_timestamp()|Volatile write through safe wrapper|
|Interrupt Control|Safe|enable_interrupt()|Masked volatile operations|

The `Rtc` struct implements `Send` and `Sync` traits with careful safety justification, allowing the driver to be used in multi-threaded contexts while maintaining memory safety guarantees.

**Safety Boundary Implementation**

```mermaid
flowchart TD
subgraph subGraph2["Hardware Zone"]
    DEVICE_MEM["Device Memory"]
    PL031_HW["PL031 Hardware"]
end
subgraph subGraph1["Unsafe Zone"]
    UNSAFE_BLOCKS["unsafe blocks"]
    RAW_PTR["*mut Registers"]
    VOLATILE_OPS["volatile operations"]
end
subgraph subGraph0["Safe Zone"]
    SAFE_API["Safe Public Methods"]
    SAFE_REFS["&self / &mut self"]
end

RAW_PTR --> DEVICE_MEM
SAFE_API --> UNSAFE_BLOCKS
SAFE_REFS --> RAW_PTR
UNSAFE_BLOCKS --> VOLATILE_OPS
VOLATILE_OPS --> PL031_HW
```

Sources: [src/lib.rs(L51 - L60)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L51-L60) [src/lib.rs(L123 - L128)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L123-L128)

## Address Space Configuration

The hardware interface requires proper address space configuration to function correctly. The base address must point to a valid PL031 device memory region with appropriate mapping characteristics.

**Address Space Requirements**

```mermaid
flowchart TD
subgraph subGraph2["Safety Requirements"]
    EXCLUSIVE["Exclusive AccessNo other aliases"]
    VALID["Valid MappingProcess address space"]
    DEVICE["Device Memory TypeProper memory attributes"]
end
subgraph subGraph1["Physical Hardware"]
    PL031_DEV["PL031 DeviceHardware Registers"]
    MMIO_REGION["MMIO Region32-byte space"]
end
subgraph subGraph0["Virtual Address Space"]
    BASE_ADDR["Base Address(from device tree)"]
    ALIGN["4-byte AlignmentRequired"]
    MAPPING["Device Memory MappingNon-cacheable"]
end

ALIGN --> MAPPING
BASE_ADDR --> ALIGN
BASE_ADDR --> EXCLUSIVE
MAPPING --> PL031_DEV
MAPPING --> VALID
MMIO_REGION --> DEVICE
PL031_DEV --> MMIO_REGION
```

The driver expects the base address to be obtained from platform-specific sources such as device tree configuration, and requires the caller to ensure proper memory mapping with device memory characteristics.

Sources: [src/lib.rs(L47 - L60)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L47-L60)