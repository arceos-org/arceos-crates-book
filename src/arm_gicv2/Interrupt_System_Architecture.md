# Interrupt System Architecture

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/README.md)
> * [src/lib.rs](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/src/lib.rs)

This document explains the interrupt classification and architectural design of the ARM GICv2 interrupt controller as modeled by the `arm_gicv2` crate. It covers the three interrupt types (SGI, PPI, SPI), their ID ranges, trigger modes, and the translation mechanisms that map logical interrupt identifiers to physical GIC interrupt IDs.

For details about the hardware interface implementation that manages these interrupts, see [Hardware Interface Implementation](/arceos-org/arm_gicv2/3-hardware-interface-implementation). For specific build and development information, see [Development Guide](/arceos-org/arm_gicv2/4-development-guide).

## GICv2 Interrupt Hierarchy

The ARM Generic Interrupt Controller version 2 supports up to 1024 interrupt sources, organized into three distinct categories based on their scope and routing capabilities. The `arm_gicv2` crate models this architecture through a combination of range constants, enumeration types, and translation functions.

#### GICv2 Interrupt Address Space

```

```

Sources: [src/lib.rs(L12 - L30)&emsp;](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/src/lib.rs#L12-L30)

## Interrupt Classification System

The crate defines three interrupt types through the `InterruptType` enum, each corresponding to different use cases and routing behaviors in the GICv2 architecture.

#### Interrupt Type Definitions

|Interrupt Type|ID Range|Constant|Purpose|
| --- | --- | --- | --- |
|SGI (Software Generated)|0-15|SGI_RANGE|Inter-processor communication|
|PPI (Private Peripheral)|16-31|PPI_RANGE|Single-processor specific interrupts|
|SPI (Shared Peripheral)|32-1019|SPI_RANGE|Multi-processor routable interrupts|

#### Interrupt Type Architecture

```

```

Sources: [src/lib.rs(L48 - L63)&emsp;](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/src/lib.rs#L48-L63) [src/lib.rs(L12 - L27)&emsp;](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/src/lib.rs#L12-L27)

### Software Generated Interrupts (SGI)

SGIs occupy interrupt IDs 0-15 and are generated through software writes to the `GICD_SGIR` register. These interrupts enable communication between processor cores in multi-core systems.

* **Range**: Defined by `SGI_RANGE` constant as `0..16`
* **Generation**: Software write to GIC distributor register
* **Routing**: Can target specific CPUs or CPU groups
* **Use Cases**: Inter-processor synchronization, cross-core notifications

### Private Peripheral Interrupts (PPI)

PPIs use interrupt IDs 16-31 and are associated with peripherals private to individual processor cores, such as per-core timers and performance monitoring units.

* **Range**: Defined by `PPI_RANGE` constant as `16..32`
* **Scope**: Private to individual CPU cores
* **Routing**: Cannot be routed between cores
* **Use Cases**: Local timers, CPU-specific peripherals

### Shared Peripheral Interrupts (SPI)

SPIs occupy the largest address space (32-1019) and represent interrupts from shared system peripherals that can be routed to any available processor core.

* **Range**: Defined by `SPI_RANGE` constant as `32..1020`
* **Routing**: Configurable to any CPU or CPU group
* **Distribution**: Managed by GIC distributor
* **Use Cases**: External device interrupts, system peripherals

Sources: [src/lib.rs(L12 - L27)&emsp;](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/src/lib.rs#L12-L27)

## ID Translation Architecture

The `translate_irq` function provides a mapping mechanism between logical interrupt identifiers and physical GIC interrupt IDs, enabling type-safe interrupt management.

#### Translation Function Flow

```

```

### Translation Examples

|Input Type|Logical ID|Physical GIC ID|Calculation|
| --- | --- | --- | --- |
|SGI|5|5|Direct mapping|
|PPI|3|19|3 + 16 (PPI_RANGE.start)|
|SPI|100|132|100 + 32 (SPI_RANGE.start)|

The function returns `Option<usize>`, with `None` indicating an invalid logical ID for the specified interrupt type.

Sources: [src/lib.rs(L65 - L90)&emsp;](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/src/lib.rs#L65-L90)

## Trigger Mode System

The GICv2 controller supports two trigger modes for interrupt signals, modeled through the `TriggerMode` enum. These modes determine how the hardware interprets interrupt signal transitions.

#### Trigger Mode Definitions

```

```

### Edge-Triggered Mode

Edge-triggered interrupts are asserted on detection of a rising edge and remain asserted until explicitly cleared by software, regardless of the ongoing signal state.

* **Enum Value**: `TriggerMode::Edge = 0`
* **Detection**: Rising edge of interrupt signal
* **Persistence**: Remains asserted until software clears
* **Use Cases**: Event-driven interrupts, completion notifications

### Level-Sensitive Mode

Level-sensitive interrupts are asserted whenever the interrupt signal is at an active level and automatically deasserted when the signal becomes inactive.

* **Enum Value**: `TriggerMode::Level = 1`
* **Detection**: Active signal level
* **Persistence**: Follows signal state
* **Use Cases**: Status-driven interrupts, continuous monitoring

Sources: [src/lib.rs(L32 - L46)&emsp;](https://github.com/arceos-org/arm_gicv2/blob/cf756f76/src/lib.rs#L32-L46)