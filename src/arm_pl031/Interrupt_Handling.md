# Interrupt Handling

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs)

This page covers the interrupt handling capabilities of the ARM PL031 RTC driver, including the interrupt registers, match-based interrupt generation, status checking, and interrupt management functions. The interrupt system allows applications to receive notifications when the RTC reaches a specific timestamp.

For general driver architecture and register operations, see [3.1](/arceos-org/arm_pl031/3.1-driver-architecture-and-design) and [3.3](/arceos-org/arm_pl031/3.3-register-operations). For memory safety considerations related to interrupt handling, see [3.5](/arceos-org/arm_pl031/3.5-memory-safety-and-concurrency).

## Interrupt System Overview

The PL031 RTC interrupt system is based on a timestamp matching mechanism. When the current RTC time matches a pre-configured match value, an interrupt is generated if enabled. The driver provides safe abstractions over the hardware interrupt registers to manage this functionality.

```mermaid
flowchart TD
subgraph subGraph3["Hardware Logic"]
    COMPARE["Time Comparison Logic"]
    INT_GEN["Interrupt Generation"]
end
subgraph subGraph2["Hardware Registers"]
    MR["Match Register (mr)"]
    IMSC["Interrupt Mask (imsc)"]
    RIS["Raw Interrupt Status (ris)"]
    MIS["Masked Interrupt Status (mis)"]
    ICR["Interrupt Clear (icr)"]
    DR["Data Register (dr)"]
end
subgraph subGraph1["Driver API"]
    SET_MATCH["set_match_timestamp()"]
    ENABLE_INT["enable_interrupt()"]
    CHECK_PENDING["interrupt_pending()"]
    CHECK_MATCHED["matched()"]
    CLEAR_INT["clear_interrupt()"]
end
subgraph subGraph0["Application Layer"]
    APP["Application Code"]
    HANDLER["Interrupt Handler"]
end

APP --> ENABLE_INT
APP --> SET_MATCH
CHECK_MATCHED --> RIS
CHECK_PENDING --> MIS
CLEAR_INT --> ICR
COMPARE --> RIS
DR --> COMPARE
ENABLE_INT --> IMSC
HANDLER --> CHECK_PENDING
HANDLER --> CLEAR_INT
IMSC --> INT_GEN
INT_GEN --> MIS
MR --> COMPARE
RIS --> INT_GEN
SET_MATCH --> MR
```

Sources: [src/lib.rs(L17 - L39)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L17-L39) [src/lib.rs(L78 - L120)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L78-L120)

## Interrupt Registers Layout

The PL031 interrupt system uses five dedicated registers within the `Registers` structure to manage interrupt functionality:

|Register|Offset|Size|Purpose|Access|
| --- | --- | --- | --- | --- |
|mr|0x04|32-bit|Match Register - stores target timestamp|Read/Write|
|imsc|0x10|8-bit|Interrupt Mask Set/Clear - enables/disables interrupts|Read/Write|
|ris|0x14|8-bit|Raw Interrupt Status - shows match status regardless of mask|Read|
|mis|0x18|8-bit|Masked Interrupt Status - shows actual interrupt state|Read|
|icr|0x1C|8-bit|Interrupt Clear Register - clears pending interrupts|Write|

```mermaid
flowchart TD
subgraph subGraph4["Driver Methods"]
    SET_MATCH_FUNC["set_match_timestamp()"]
    ENABLE_FUNC["enable_interrupt()"]
    MATCHED_FUNC["matched()"]
    PENDING_FUNC["interrupt_pending()"]
    CLEAR_FUNC["clear_interrupt()"]
end
subgraph subGraph3["Register Operations"]
    subgraph subGraph2["Interrupt Clearing"]
        ICR_OP["icr: write_volatile(0x01)"]
    end
    subgraph subGraph1["Status Reading"]
        RIS_OP["ris: read_volatile() & 0x01"]
        MIS_OP["mis: read_volatile() & 0x01"]
    end
    subgraph Configuration["Configuration"]
        MR_OP["mr: write_volatile(timestamp)"]
        IMSC_OP["imsc: write_volatile(0x01/0x00)"]
    end
end

CLEAR_FUNC --> ICR_OP
ENABLE_FUNC --> IMSC_OP
MATCHED_FUNC --> RIS_OP
PENDING_FUNC --> MIS_OP
SET_MATCH_FUNC --> MR_OP
```

Sources: [src/lib.rs(L17 - L39)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L17-L39) [src/lib.rs(L78 - L120)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L78-L120)

## Interrupt Lifecycle and State Management

The interrupt system follows a well-defined lifecycle from configuration through interrupt generation to clearing:

```mermaid
stateDiagram-v2
[*] --> Idle : "RTC initialized"
Idle --> Configured : "set_match_timestamp()"
Configured --> Armed : "enable_interrupt(true)"
Armed --> Matched : "RTC time == match time"
Matched --> InterruptPending : "Hardware generates interrupt"
InterruptPending --> Cleared : "clear_interrupt()"
Cleared --> Armed : "Interrupt cleared, still armed"
Armed --> Disabled : "enable_interrupt(false)"
Disabled --> Idle : "Interrupt disabled"
note left of Matched : ['"matched() returns true"']
note left of InterruptPending : ['"interrupt_pending() returns true"']
note left of Cleared : ['"Hardware interrupt cleared"']
```

### Interrupt Configuration

The `set_match_timestamp()` method configures when an interrupt should be generated:

```rust
pub fn set_match_timestamp(&mut self, match_timestamp: u32)
```

This function writes the target timestamp to the match register (`mr`). When the RTC's data register (`dr`) equals this value, the hardware sets the raw interrupt status bit.

### Interrupt Enabling and Masking

The `enable_interrupt()` method controls whether interrupts are actually generated:

```rust
pub fn enable_interrupt(&mut self, mask: bool)
```

When `mask` is `true`, the function writes `0x01` to the interrupt mask register (`imsc`), enabling interrupts. When `false`, it writes `0x00`, disabling them.

Sources: [src/lib.rs(L78 - L82)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L78-L82) [src/lib.rs(L108 - L113)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L108-L113)

## Status Checking and Interrupt Detection

The driver provides two distinct methods for checking interrupt status, each serving different purposes:

### Raw Match Status (matched())

The `matched()` method checks the raw interrupt status register (`ris`) to determine if the match condition has occurred, regardless of whether interrupts are enabled:

```rust
pub fn matched(&self) -> bool
```

This method reads the `ris` register and returns `true` if bit 0 is set, indicating that the RTC time matches the match register value.

### Masked Interrupt Status (interrupt_pending())

The `interrupt_pending()` method checks the masked interrupt status register (`mis`) to determine if there is an actual pending interrupt:

```rust
pub fn interrupt_pending(&self) -> bool
```

This method returns `true` only when both the match condition is met AND interrupts are enabled. This is the method typically used in interrupt handlers.

```mermaid
flowchart TD
subgraph subGraph0["Status Check Flow"]
    TIME_CHECK["RTC Time == Match Time?"]
    RIS_SET["Set ris bit 0"]
    MASK_CHECK["Interrupt enabled (imsc)?"]
    MIS_SET["Set mis bit 0"]
    MATCHED_CALL["matched() called"]
    READ_RIS["Read ris register"]
    MATCHED_RESULT["Return (ris & 0x01) != 0"]
    PENDING_CALL["interrupt_pending() called"]
    READ_MIS["Read mis register"]
    PENDING_RESULT["Return (mis & 0x01) != 0"]
end

MASK_CHECK --> MIS_SET
MATCHED_CALL --> READ_RIS
PENDING_CALL --> READ_MIS
READ_MIS --> PENDING_RESULT
READ_RIS --> MATCHED_RESULT
RIS_SET --> MASK_CHECK
TIME_CHECK --> RIS_SET
```

Sources: [src/lib.rs(L86 - L91)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L86-L91) [src/lib.rs(L97 - L102)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L97-L102)

## Interrupt Clearing

The `clear_interrupt()` method resets the interrupt state by writing to the interrupt clear register (`icr`):

```rust
pub fn clear_interrupt(&mut self)
```

This function writes `0x01` to the `icr` register, which clears both the raw interrupt status (`ris`) and masked interrupt status (`mis`) bits. After clearing, the interrupt system can generate new interrupts when the next match condition occurs.

## Usage Patterns and Best Practices

### Basic Interrupt Setup

```
// Configure interrupt for specific timestamp
rtc.set_match_timestamp(target_time);
rtc.enable_interrupt(true);
```

### Interrupt Handler Pattern

```
// In interrupt handler
if rtc.interrupt_pending() {
    // Handle the interrupt
    handle_rtc_interrupt();
    
    // Clear the interrupt
    rtc.clear_interrupt();
    
    // Optionally set new match time
    rtc.set_match_timestamp(next_target_time);
}
```

### Polling Pattern (Alternative to Interrupts)

```
// Poll for match without using interrupts
rtc.enable_interrupt(false);
loop {
    if rtc.matched() {
        handle_rtc_event();
        // Set new match time or break
        rtc.set_match_timestamp(next_target_time);
    }
    // Other work...
}
```

Sources: [src/lib.rs(L78 - L120)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L78-L120)

## Thread Safety and Concurrency

The interrupt-related methods maintain the same thread safety guarantees as the rest of the driver. Reading methods like `matched()` and `interrupt_pending()` are safe to call concurrently, while writing methods like `set_match_timestamp()`, `enable_interrupt()`, and `clear_interrupt()` require mutable access to ensure atomicity.

The driver implements `Send` and `Sync` traits, allowing the RTC instance to be shared across threads with appropriate synchronization mechanisms.

Sources: [src/lib.rs(L123 - L128)&emsp;](https://github.com/arceos-org/arm_pl031/blob/8cc6761d/src/lib.rs#L123-L128)