# Hardware Implementations

> **Relevant source files**
> * [axdriver_net/src/fxmac.rs](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/fxmac.rs)
> * [axdriver_net/src/ixgbe.rs](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/ixgbe.rs)

This document covers the concrete network hardware driver implementations that provide device-specific functionality within the axdriver network subsystem. These implementations demonstrate how the abstract `NetDriverOps` trait is realized for specific network controllers, including buffer management strategies and hardware abstraction layer integration.

For information about the network driver interface abstractions, see [Network Driver Interface](/arceos-org/axdriver_crates/4.1-network-driver-interface). For details about buffer management patterns, see [Network Buffer Management](/arceos-org/axdriver_crates/4.2-network-buffer-management).

## Overview

The axdriver network subsystem includes two primary hardware implementations that showcase different architectural approaches:

|Driver|Hardware Target|Buffer Strategy|External Dependencies|
| --- | --- | --- | --- |
|FXmacNic|PhytiumPi Ethernet Controller|Queue-based RX buffering|fxmac_rscrate|
|IxgbeNic|Intel 10GbE Controller|Memory pool allocation|ixgbe-drivercrate|

Both implementations provide the same `NetDriverOps` interface while handling hardware-specific details internally.

## FXmac Network Controller Implementation

### Architecture and Integration

The `FXmacNic` struct provides an implementation for PhytiumPi Ethernet controllers using a queue-based approach for receive buffer management.

```mermaid
flowchart TD
subgraph subGraph2["Trait Implementation"]
    BASE_OPS["BaseDriverOps"]
    NET_OPS["NetDriverOps"]
    DEVICE_NAME["device_name()"]
    MAC_ADDR["mac_address()"]
    RECEIVE["receive()"]
    TRANSMIT["transmit()"]
end
subgraph subGraph1["External Hardware Layer"]
    FXMAC_RS["fxmac_rs crate"]
    XMAC_INIT["xmac_init()"]
    GET_MAC["FXmacGetMacAddress()"]
    RECV_HANDLER["FXmacRecvHandler()"]
    TX_FUNC["FXmacLwipPortTx()"]
end
subgraph subGraph0["FXmacNic Driver Structure"]
    FXMAC_STRUCT["FXmacNic"]
    INNER["inner: &'static mut FXmac"]
    HWADDR["hwaddr: [u8; 6]"]
    RX_QUEUE["rx_buffer_queue: VecDeque"]
end

BASE_OPS --> DEVICE_NAME
FXMAC_STRUCT --> BASE_OPS
FXMAC_STRUCT --> HWADDR
FXMAC_STRUCT --> INNER
FXMAC_STRUCT --> NET_OPS
FXMAC_STRUCT --> RX_QUEUE
HWADDR --> GET_MAC
INNER --> XMAC_INIT
NET_OPS --> MAC_ADDR
NET_OPS --> RECEIVE
NET_OPS --> TRANSMIT
RECEIVE --> RECV_HANDLER
TRANSMIT --> TX_FUNC
```

Sources: [axdriver_net/src/fxmac.rs(L1 - L145)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/fxmac.rs#L1-L145)

### Initialization and Hardware Integration

The `FXmacNic::init()` function demonstrates the initialization pattern for hardware-dependent drivers:

```mermaid
flowchart TD
subgraph subGraph0["Initialization Sequence"]
    MAPPED_REGS["mapped_regs: usize"]
    INIT_QUEUE["VecDeque::with_capacity(QS)"]
    GET_MAC_ADDR["FXmacGetMacAddress(&mut hwaddr, 0)"]
    XMAC_INIT_CALL["xmac_init(&hwaddr)"]
    CONSTRUCT["FXmacNic { inner, hwaddr, rx_buffer_queue }"]
end

GET_MAC_ADDR --> XMAC_INIT_CALL
INIT_QUEUE --> GET_MAC_ADDR
MAPPED_REGS --> INIT_QUEUE
XMAC_INIT_CALL --> CONSTRUCT
```

The driver uses a fixed queue size of 64 (`QS = 64`) for both receive and transmit operations, defined as a constant.

Sources: [axdriver_net/src/fxmac.rs(L16)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/fxmac.rs#L16-L16) [axdriver_net/src/fxmac.rs(L30 - L45)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/fxmac.rs#L30-L45)

### Buffer Management Strategy

The FXmac implementation uses a simple queue-based approach for managing received packets:

* **Receive Path**: Uses `VecDeque<NetBufPtr>` to queue incoming packets from `FXmacRecvHandler()`
* **Transmit Path**: Allocates `Box<Vec<u8>>` for each transmission
* **Buffer Recycling**: Drops allocated boxes directly using `Box::from_raw()`

The `receive()` method demonstrates this pattern by first checking the local queue, then polling the hardware.

Sources: [axdriver_net/src/fxmac.rs(L92 - L118)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/fxmac.rs#L92-L118) [axdriver_net/src/fxmac.rs(L134 - L144)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/fxmac.rs#L134-L144)

## Intel ixgbe Network Controller Implementation

### Architecture and Memory Pool Integration

The `IxgbeNic` struct implements a more sophisticated memory management approach using pre-allocated memory pools:

```mermaid
flowchart TD
subgraph subGraph2["Generic Parameters"]
    HAL_PARAM["H: IxgbeHal"]
    QS_PARAM["QS: queue size"]
    QN_PARAM["QN: queue number"]
end
subgraph subGraph1["External Hardware Layer"]
    IXGBE_DRIVER["ixgbe-driver crate"]
    IXGBE_DEVICE["IxgbeDevice"]
    MEM_POOL_TYPE["MemPool"]
    IXGBE_NETBUF["IxgbeNetBuf"]
    NIC_DEVICE["NicDevice trait"]
end
subgraph subGraph0["IxgbeNic Driver Structure"]
    IXGBE_STRUCT["IxgbeNic"]
    INNER_DEV["inner: IxgbeDevice"]
    MEM_POOL["mem_pool: Arc"]
    RX_QUEUE["rx_buffer_queue: VecDeque"]
end

INNER_DEV --> IXGBE_DEVICE
IXGBE_DEVICE --> IXGBE_NETBUF
IXGBE_DEVICE --> NIC_DEVICE
IXGBE_STRUCT --> HAL_PARAM
IXGBE_STRUCT --> INNER_DEV
IXGBE_STRUCT --> MEM_POOL
IXGBE_STRUCT --> QN_PARAM
IXGBE_STRUCT --> QS_PARAM
IXGBE_STRUCT --> RX_QUEUE
MEM_POOL --> MEM_POOL_TYPE
```

Sources: [axdriver_net/src/ixgbe.rs(L18 - L28)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/ixgbe.rs#L18-L28)

### Memory Pool Configuration

The ixgbe driver uses predefined memory pool parameters for efficient buffer allocation:

|Parameter|Value|Purpose|
| --- | --- | --- |
|MEM_POOL|4096|Total memory pool entries|
|MEM_POOL_ENTRY_SIZE|2048|Size per pool entry in bytes|
|RX_BUFFER_SIZE|1024|Receive buffer queue capacity|
|RECV_BATCH_SIZE|64|Batch size for receive operations|

These constants ensure optimal performance for high-throughput network operations.

Sources: [axdriver_net/src/ixgbe.rs(L13 - L16)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/ixgbe.rs#L13-L16)

### NetBufPtr Conversion Pattern

The ixgbe implementation demonstrates a sophisticated buffer conversion pattern between `IxgbeNetBuf` and `NetBufPtr`:

```mermaid
flowchart TD
subgraph subGraph0["Buffer Conversion Flow"]
    IXGBE_BUF["IxgbeNetBuf"]
    MANUALLY_DROP["ManuallyDrop::new(buf)"]
    PACKET_PTR["buf.packet_mut().as_mut_ptr()"]
    NET_BUF_PTR["NetBufPtr::new(raw_ptr, buf_ptr, len)"]
    subgraph subGraph1["Reverse Conversion"]
        NET_BUF_PTR_REV["NetBufPtr"]
        CONSTRUCT["IxgbeNetBuf::construct()"]
        IXGBE_BUF_REV["IxgbeNetBuf"]
        POOL_ENTRY["buf.pool_entry() as *mut u8"]
    end
end

CONSTRUCT --> IXGBE_BUF_REV
IXGBE_BUF --> MANUALLY_DROP
MANUALLY_DROP --> PACKET_PTR
MANUALLY_DROP --> POOL_ENTRY
NET_BUF_PTR_REV --> CONSTRUCT
PACKET_PTR --> NET_BUF_PTR
POOL_ENTRY --> NET_BUF_PTR
```

The `From<IxgbeNetBuf>` implementation uses `ManuallyDrop` to avoid premature deallocation while transferring ownership to the `NetBufPtr` abstraction.

Sources: [axdriver_net/src/ixgbe.rs(L143 - L162)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/ixgbe.rs#L143-L162)

## Trait Implementation Patterns

### BaseDriverOps Implementation

Both drivers implement `BaseDriverOps` with device-specific identification:

|Driver|device_name()Return Value|Source|
| --- | --- | --- |
|FXmacNic|"cdns,phytium-gem-1.0"|Hardware device tree compatible string|
|IxgbeNic|self.inner.get_driver_name()|Delegated to underlying driver|

Sources: [axdriver_net/src/fxmac.rs(L48 - L56)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/fxmac.rs#L48-L56) [axdriver_net/src/ixgbe.rs(L50 - L58)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/ixgbe.rs#L50-L58)

### NetDriverOps Core Methods

The receive and transmit implementations showcase different architectural approaches:

```mermaid
flowchart TD
subgraph subGraph1["Ixgbe Receive Pattern"]
    IXGBE_CAN_RECV["can_receive(): !queue.is_empty() || inner.can_receive(0)"]
    IXGBE_RECV_CHECK["Check local queue first"]
    IXGBE_BATCH_RECV["inner.receive_packets(0, RECV_BATCH_SIZE, closure)"]
    IXGBE_QUEUE_PUSH["Closure pushes to rx_buffer_queue"]
    FXMAC_CAN_RECV["can_receive(): !rx_buffer_queue.is_empty()"]
    FXMAC_RECV_CHECK["Check local queue first"]
    FXMAC_HW_POLL["FXmacRecvHandler(self.inner)"]
    FXMAC_QUEUE_PUSH["Push to rx_buffer_queue"]
end
subgraph subGraph0["FXmac Receive Pattern"]
    IXGBE_CAN_RECV["can_receive(): !queue.is_empty() || inner.can_receive(0)"]
    IXGBE_RECV_CHECK["Check local queue first"]
    IXGBE_BATCH_RECV["inner.receive_packets(0, RECV_BATCH_SIZE, closure)"]
    IXGBE_QUEUE_PUSH["Closure pushes to rx_buffer_queue"]
    FXMAC_CAN_RECV["can_receive(): !rx_buffer_queue.is_empty()"]
    FXMAC_RECV_CHECK["Check local queue first"]
    FXMAC_HW_POLL["FXmacRecvHandler(self.inner)"]
    FXMAC_QUEUE_PUSH["Push to rx_buffer_queue"]
end

FXMAC_CAN_RECV --> FXMAC_RECV_CHECK
FXMAC_HW_POLL --> FXMAC_QUEUE_PUSH
FXMAC_RECV_CHECK --> FXMAC_HW_POLL
IXGBE_BATCH_RECV --> IXGBE_QUEUE_PUSH
IXGBE_CAN_RECV --> IXGBE_RECV_CHECK
IXGBE_RECV_CHECK --> IXGBE_BATCH_RECV
```

Sources: [axdriver_net/src/fxmac.rs(L71 - L118)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/fxmac.rs#L71-L118) [axdriver_net/src/ixgbe.rs(L73 - L124)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/ixgbe.rs#L73-L124)

## Hardware Abstraction Layer Integration

### External Crate Dependencies

Both implementations demonstrate integration with external hardware abstraction crates:

|Driver|External Crate|Key Functions Used|
| --- | --- | --- |
|FXmacNic|fxmac_rs|xmac_init,FXmacGetMacAddress,FXmacRecvHandler,FXmacLwipPortTx|
|IxgbeNic|ixgbe-driver|IxgbeDevice::init,MemPool::allocate,receive_packets,send|

This pattern allows the axdriver framework to leverage existing hardware-specific implementations while providing a unified interface.

Sources: [axdriver_net/src/fxmac.rs(L7)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/fxmac.rs#L7-L7) [axdriver_net/src/ixgbe.rs(L6 - L7)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/ixgbe.rs#L6-L7)

### Error Handling Translation

Both drivers translate hardware-specific errors to the common `DevError` type:

```mermaid
flowchart TD
subgraph subGraph1["Ixgbe Error Translation"]
    IXGBE_ERR["IxgbeError"]
    IXGBE_NOT_READY["IxgbeError::NotReady"]
    IXGBE_QUEUE_FULL["IxgbeError::QueueFull"]
    DEV_AGAIN["DevError::Again"]
    DEV_BAD_STATE["DevError::BadState"]
    FXMAC_RET["FXmacLwipPortTx return value"]
    FXMAC_CHECK["ret < 0"]
    FXMAC_ERROR["DevError::Again"]
end
subgraph subGraph0["FXmac Error Translation"]
    IXGBE_NOT_READY["IxgbeError::NotReady"]
    DEV_AGAIN["DevError::Again"]
    FXMAC_RET["FXmacLwipPortTx return value"]
    FXMAC_CHECK["ret < 0"]
    FXMAC_ERROR["DevError::Again"]
end

FXMAC_CHECK --> FXMAC_ERROR
FXMAC_RET --> FXMAC_CHECK
IXGBE_ERR --> DEV_BAD_STATE
IXGBE_ERR --> IXGBE_NOT_READY
IXGBE_ERR --> IXGBE_QUEUE_FULL
IXGBE_NOT_READY --> DEV_AGAIN
IXGBE_QUEUE_FULL --> DEV_AGAIN
```

Sources: [axdriver_net/src/fxmac.rs(L127 - L131)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/fxmac.rs#L127-L131) [axdriver_net/src/ixgbe.rs(L118 - L122)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/ixgbe.rs#L118-L122) [axdriver_net/src/ixgbe.rs(L130 - L134)&emsp;](https://github.com/arceos-org/axdriver_crates/blob/84eb2170/axdriver_net/src/ixgbe.rs#L130-L134)