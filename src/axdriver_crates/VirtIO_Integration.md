# VirtIO Integration

> **Relevant source files**
> * [axdriver_virtio/Cargo.toml](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_virtio/Cargo.toml)
> * [axdriver_virtio/src/lib.rs](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_virtio/src/lib.rs)

The VirtIO integration layer provides a bridge between the external `virtio-drivers` crate and the axdriver framework, enabling virtualized hardware devices to be accessed through the same standardized interfaces as physical hardware. This integration allows ArceOS to run efficiently in virtualized environments while maintaining consistent driver APIs.

For information about the underlying driver traits that VirtIO devices implement, see [Foundation Layer (axdriver_base)](/arceos-org/axdriver_crates/3-foundation-layer-(axdriver_base)). For details on specific device type implementations, see [Network Drivers](/arceos-org/axdriver_crates/4-network-drivers), [Block Storage Drivers](/arceos-org/axdriver_crates/5-block-storage-drivers), and [Display Drivers](/arceos-org/axdriver_crates/6-display-drivers).

## Architecture Overview

The VirtIO integration follows an adapter pattern that wraps `virtio-drivers` device implementations to conform to the axdriver trait system. This design allows the same application code to work with both physical and virtualized hardware.

```mermaid
flowchart TD
subgraph Target_Traits["axdriver Traits"]
    BASE_OPS["BaseDriverOps"]
    BLOCK_OPS["BlockDriverOps"]
    NET_OPS["NetDriverOps"]
    DISPLAY_OPS["DisplayDriverOps"]
end
subgraph Bridge_Layer["axdriver_virtio Bridge"]
    PROBE_MMIO["probe_mmio_device()"]
    PROBE_PCI["probe_pci_device()"]
    TYPE_CONV["as_dev_type()"]
    ERR_CONV["as_dev_err()"]
    subgraph Device_Wrappers["Device Wrappers"]
        VIRTIO_BLK["VirtIoBlkDev"]
        VIRTIO_NET["VirtIoNetDev"]
        VIRTIO_GPU["VirtIoGpuDev"]
    end
end
subgraph External_VirtIO["External VirtIO Layer"]
    VD["virtio-drivers crate"]
    TRANSPORT["Transport Layer"]
    HAL["VirtIoHal trait"]
end

ERR_CONV --> BASE_OPS
PROBE_MMIO --> TYPE_CONV
PROBE_PCI --> TYPE_CONV
TRANSPORT --> PROBE_MMIO
TRANSPORT --> PROBE_PCI
TYPE_CONV --> VIRTIO_BLK
TYPE_CONV --> VIRTIO_GPU
TYPE_CONV --> VIRTIO_NET
VD --> PROBE_MMIO
VD --> PROBE_PCI
VIRTIO_BLK --> BASE_OPS
VIRTIO_BLK --> BLOCK_OPS
VIRTIO_GPU --> BASE_OPS
VIRTIO_GPU --> DISPLAY_OPS
VIRTIO_NET --> BASE_OPS
VIRTIO_NET --> NET_OPS
```

**Sources:** [axdriver_virtio/src/lib.rs(L1 - L98)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_virtio/src/lib.rs#L1-L98)

## Transport Layer Support

The VirtIO integration supports both MMIO and PCI transport mechanisms, providing flexibility for different virtualization platforms and hardware configurations.

### MMIO Transport

The `probe_mmio_device` function discovers VirtIO devices mapped into memory regions, commonly used in embedded virtualization scenarios.

```mermaid
flowchart TD
subgraph Detection_Process["Detection Process"]
    PROBE_FUNC["probe_mmio_device()"]
    DEVICE_TYPE["DeviceType"]
    TRANSPORT_OBJ["Transport Object"]
end
subgraph MMIO_Discovery["MMIO Device Discovery"]
    MEM_REGION["Memory Regionreg_base: *mut u8"]
    VIRTIO_HEADER["VirtIOHeader"]
    MMIO_TRANSPORT["MmioTransport"]
end

MEM_REGION --> VIRTIO_HEADER
MMIO_TRANSPORT --> PROBE_FUNC
PROBE_FUNC --> DEVICE_TYPE
PROBE_FUNC --> TRANSPORT_OBJ
VIRTIO_HEADER --> MMIO_TRANSPORT
```

The function performs device validation and type identification:

|Step|Operation|Result|
| --- | --- | --- |
|1|CreateNonNull<VirtIOHeader>from memory base|Header pointer validation|
|2|InitializeMmioTransport|Transport layer setup|
|3|Callas_dev_type()on device type|Convert toDeviceType|
|4|Return tuple|(DeviceType, MmioTransport)|

**Sources:** [axdriver_virtio/src/lib.rs(L38 - L53)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_virtio/src/lib.rs#L38-L53)

### PCI Transport

The `probe_pci_device` function discovers VirtIO devices on the PCI bus, typical for desktop and server virtualization environments.

```mermaid
flowchart TD
subgraph Probing_Logic["Probing Logic"]
    VIRTIO_TYPE["virtio_device_type()"]
    TYPE_CONV["as_dev_type()"]
    PCI_TRANSPORT["PciTransport::new()"]
end
subgraph PCI_Discovery["PCI Device Discovery"]
    PCI_ROOT["PciRoot"]
    BDF["DeviceFunction(Bus/Device/Function)"]
    DEV_INFO["DeviceFunctionInfo"]
end

BDF --> PCI_TRANSPORT
DEV_INFO --> VIRTIO_TYPE
PCI_ROOT --> PCI_TRANSPORT
TYPE_CONV --> PCI_TRANSPORT
VIRTIO_TYPE --> TYPE_CONV
```

**Sources:** [axdriver_virtio/src/lib.rs(L55 - L69)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_virtio/src/lib.rs#L55-L69)

## Device Type Mapping

The integration includes a comprehensive type conversion system that maps VirtIO device types to axdriver device categories.

### Type Conversion Function

The `as_dev_type` function provides the core mapping between VirtIO and axdriver type systems:

```mermaid
flowchart TD
subgraph Axdriver_Types["axdriver Device Types"]
    DEV_BLOCK["DeviceType::Block"]
    DEV_NET["DeviceType::Net"]
    DEV_DISPLAY["DeviceType::Display"]
    DEV_NONE["None"]
end
subgraph Mapping_Function["as_dev_type()"]
    MATCH_EXPR["match expression"]
end
subgraph VirtIO_Types["VirtIO Device Types"]
    VIRTIO_BLOCK["VirtIoDevType::Block"]
    VIRTIO_NET["VirtIoDevType::Network"]
    VIRTIO_GPU["VirtIoDevType::GPU"]
    VIRTIO_OTHER["Other VirtIO Types"]
end

MATCH_EXPR --> DEV_BLOCK
MATCH_EXPR --> DEV_DISPLAY
MATCH_EXPR --> DEV_NET
MATCH_EXPR --> DEV_NONE
VIRTIO_BLOCK --> MATCH_EXPR
VIRTIO_GPU --> MATCH_EXPR
VIRTIO_NET --> MATCH_EXPR
VIRTIO_OTHER --> MATCH_EXPR
```

|VirtIO Type|axdriver Type|Purpose|
| --- | --- | --- |
|Block|DeviceType::Block|Storage devices|
|Network|DeviceType::Net|Network interfaces|
|GPU|DeviceType::Display|Graphics and display|
|Other|None|Unsupported device types|

**Sources:** [axdriver_virtio/src/lib.rs(L71 - L79)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_virtio/src/lib.rs#L71-L79)

## Error Handling Integration

The VirtIO integration includes comprehensive error mapping to ensure consistent error handling across the driver framework.

### Error Conversion System

The `as_dev_err` function maps `virtio_drivers::Error` variants to `axdriver_base::DevError` types:

```mermaid
flowchart TD
subgraph Axdriver_Errors["axdriver_base::DevError"]
    BAD_STATE["BadState"]
    AGAIN["Again"]
    ALREADY_EXISTS["AlreadyExists"]
    INVALID_PARAM_OUT["InvalidParam"]
    NO_MEMORY["NoMemory"]
    IO_OUT["Io"]
    UNSUPPORTED_OUT["Unsupported"]
end
subgraph Conversion["as_dev_err()"]
    ERROR_MATCH["match expression"]
end
subgraph VirtIO_Errors["virtio_drivers::Error"]
    QUEUE_FULL["QueueFull"]
    NOT_READY["NotReady"]
    WRONG_TOKEN["WrongToken"]
    ALREADY_USED["AlreadyUsed"]
    INVALID_PARAM["InvalidParam"]
    DMA_ERROR["DmaError"]
    IO_ERROR["IoError"]
    UNSUPPORTED["Unsupported"]
    CONFIG_ERRORS["Config Errors"]
    OTHER_ERRORS["Other Errors"]
end

ALREADY_USED --> ERROR_MATCH
CONFIG_ERRORS --> ERROR_MATCH
DMA_ERROR --> ERROR_MATCH
ERROR_MATCH --> AGAIN
ERROR_MATCH --> ALREADY_EXISTS
ERROR_MATCH --> BAD_STATE
ERROR_MATCH --> INVALID_PARAM_OUT
ERROR_MATCH --> IO_OUT
ERROR_MATCH --> NO_MEMORY
ERROR_MATCH --> UNSUPPORTED_OUT
INVALID_PARAM --> ERROR_MATCH
IO_ERROR --> ERROR_MATCH
NOT_READY --> ERROR_MATCH
OTHER_ERRORS --> ERROR_MATCH
QUEUE_FULL --> ERROR_MATCH
UNSUPPORTED --> ERROR_MATCH
WRONG_TOKEN --> ERROR_MATCH
```

**Sources:** [axdriver_virtio/src/lib.rs(L81 - L97)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_virtio/src/lib.rs#L81-L97)

## Feature-Based Compilation

The VirtIO integration uses feature flags to enable selective compilation of device types, allowing minimal builds for specific deployment scenarios.

### Feature Configuration

|Feature|Dependencies|Enabled Components|
| --- | --- | --- |
|block|axdriver_block|VirtIoBlkDevwrapper|
|net|axdriver_net|VirtIoNetDevwrapper|
|gpu|axdriver_display|VirtIoGpuDevwrapper|

```mermaid
flowchart TD
subgraph Public_Exports["Public API"]
    PUB_BLK["pub use VirtIoBlkDev"]
    PUB_NET["pub use VirtIoNetDev"]
    PUB_GPU["pub use VirtIoGpuDev"]
end
subgraph Conditional_Modules["Conditional Compilation"]
    MOD_BLK["mod blk"]
    MOD_NET["mod net"]
    MOD_GPU["mod gpu"]
end
subgraph Feature_Flags["Cargo Features"]
    FEAT_BLOCK["block"]
    FEAT_NET["net"]
    FEAT_GPU["gpu"]
end

FEAT_BLOCK --> MOD_BLK
FEAT_GPU --> MOD_GPU
FEAT_NET --> MOD_NET
MOD_BLK --> PUB_BLK
MOD_GPU --> PUB_GPU
MOD_NET --> PUB_NET
```

**Sources:** [axdriver_virtio/Cargo.toml(L14 - L24)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_virtio/Cargo.toml#L14-L24) [axdriver_virtio/src/lib.rs(L16 - L28)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_virtio/src/lib.rs#L16-L28)

## Public API Surface

The crate re-exports essential VirtIO components to provide a complete integration interface:

### Core Re-exports

|Export|Source|Purpose|
| --- | --- | --- |
|pci|virtio_drivers::transport::pci::bus|PCI bus operations|
|MmioTransport|virtio_drivers::transport::mmio|MMIO transport|
|PciTransport|virtio_drivers::transport::pci|PCI transport|
|VirtIoHal|virtio_drivers::Hal|Hardware abstraction trait|
|PhysAddr|virtio_drivers::PhysAddr|Physical address type|
|BufferDirection|virtio_drivers::BufferDirection|DMA buffer direction|

**Sources:** [axdriver_virtio/src/lib.rs(L30 - L36)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_virtio/src/lib.rs#L30-L36)