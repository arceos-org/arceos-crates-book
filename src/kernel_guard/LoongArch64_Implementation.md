# LoongArch64 Implementation

> **Relevant source files**
> * [src/arch/loongarch64.rs](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/arch/loongarch64.rs)

## Purpose and Scope

This document covers the LoongArch64-specific implementation of interrupt control mechanisms within the kernel_guard crate. The LoongArch64 implementation provides platform-specific functions for saving, disabling, and restoring interrupt states using Control Status Register (CSR) manipulation.

For general architecture abstraction concepts, see [Architecture Abstraction Layer](/arceos-org/kernel_guard/3.1-architecture-abstraction-layer). For other architecture implementations, see [x86/x86_64 Implementation](/arceos-org/kernel_guard/3.2-x86x86_64-implementation), [RISC-V Implementation](/arceos-org/kernel_guard/3.3-risc-v-implementation), and [AArch64 Implementation](/arceos-org/kernel_guard/3.4-aarch64-implementation).

## Architecture Integration

The LoongArch64 implementation is conditionally compiled as part of the multi-architecture support system. It provides the same interface as other architectures but uses LoongArch64-specific Control Status Registers and the `csrxchg` instruction for atomic CSR exchange operations.

### Architecture Selection Flow

```mermaid
flowchart TD
CFGIF["cfg_if! macro evaluation"]
LOONG_CHECK["target_arch = loongarch64?"]
LOONG_MOD["mod loongarch64"]
OTHER_ARCH["Other architecture modules"]
PUB_USE["pub use self::loongarch64::*"]
LOCAL_IRQ_SAVE["local_irq_save_and_disable()"]
LOCAL_IRQ_RESTORE["local_irq_restore()"]
IRQ_SAVE_GUARD["IrqSave guard"]
NO_PREEMPT_IRQ_SAVE["NoPreemptIrqSave guard"]

CFGIF --> LOONG_CHECK
IRQ_SAVE_GUARD --> NO_PREEMPT_IRQ_SAVE
LOCAL_IRQ_RESTORE --> IRQ_SAVE_GUARD
LOCAL_IRQ_SAVE --> IRQ_SAVE_GUARD
LOONG_CHECK --> LOONG_MOD
LOONG_CHECK --> OTHER_ARCH
LOONG_MOD --> PUB_USE
PUB_USE --> LOCAL_IRQ_RESTORE
PUB_USE --> LOCAL_IRQ_SAVE
```

Sources: [src/arch/loongarch64.rs(L1 - L18)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/arch/loongarch64.rs#L1-L18)

## Interrupt Control Implementation

The LoongArch64 implementation centers around two core functions that manipulate the interrupt enable bit in Control Status Registers using the atomic `csrxchg` instruction.

### Core Functions

|Function|Purpose|Return Value|
| --- | --- | --- |
|local_irq_save_and_disable()|Save current interrupt state and disable interrupts|Previous interrupt enable state (masked)|
|local_irq_restore(flags)|Restore interrupt state from saved flags|None|

### CSR Manipulation Details

```mermaid
flowchart TD
subgraph local_irq_restore["local_irq_restore"]
    RESTORE_START["Function entryflags parameter"]
    CSRXCHG_RESTORE["csrxchg {}, {}, 0x0in(reg) flagsin(reg) IE_MASK"]
    RESTORE_END["Function return"]
end
subgraph local_irq_save_and_disable["local_irq_save_and_disable"]
    SAVE_START["Function entry"]
    FLAGS_INIT["flags: usize = 0"]
    CSRXCHG_SAVE["csrxchg {}, {}, 0x0inout(reg) flagsin(reg) IE_MASK"]
    MASK_RETURN["Return flags & IE_MASK"]
end
subgraph subGraph0["LoongArch64 CSR Operations"]
    IE_MASK["IE_MASK constant1 << 2 (bit 2)"]
    CSRXCHG["csrxchg instructionAtomic CSR exchange"]
end

CSRXCHG_RESTORE --> RESTORE_END
CSRXCHG_SAVE --> MASK_RETURN
FLAGS_INIT --> CSRXCHG_SAVE
IE_MASK --> CSRXCHG_RESTORE
IE_MASK --> CSRXCHG_SAVE
RESTORE_START --> CSRXCHG_RESTORE
SAVE_START --> FLAGS_INIT
```

Sources: [src/arch/loongarch64.rs(L3 - L17)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/arch/loongarch64.rs#L3-L17)

## Control Status Register (CSR) Operations

The implementation uses CSR address `0x0` (likely CRMD - Current Mode) with the `csrxchg` instruction for atomic read-modify-write operations on the interrupt enable bit.

### Interrupt Enable Bit Manipulation

The `IE_MASK` constant defines bit 2 as the interrupt enable bit:

```mermaid
flowchart TD
subgraph subGraph1["IE_MASK Constant"]
    MASK_VAL["1 << 2 = 0x4"]
    MASK_PURPOSE["Isolates IE bitfor CSR operations"]
end
subgraph subGraph0["CSR Bit Layout"]
    BIT0["Bit 0"]
    BIT1["Bit 1"]
    BIT2["Bit 2IE (Interrupt Enable)"]
    BIT3["Bit 3"]
    BITN["Bit N"]
end

BIT2 --> MASK_VAL
MASK_VAL --> MASK_PURPOSE
```

Sources: [src/arch/loongarch64.rs(L3)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/arch/loongarch64.rs#L3-L3)

### Assembly Instruction Details

The `csrxchg` instruction performs atomic exchange operations:

* **Save Operation**: `csrxchg {flags}, {IE_MASK}, 0x0` clears the IE bit and returns the old CSR value
* **Restore Operation**: `csrxchg {flags}, {IE_MASK}, 0x0` sets the IE bit based on the flags parameter

The instruction syntax uses:

* `inout(reg) flags` for read-modify-write of the flags variable
* `in(reg) IE_MASK` for the bit mask input
* `0x0` as the CSR address

Sources: [src/arch/loongarch64.rs(L9 - L16)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/arch/loongarch64.rs#L9-L16)

## Integration with Guard System

The LoongArch64 functions integrate seamlessly with the kernel_guard's RAII guard system through the architecture abstraction layer.

### Guard Usage Flow

```mermaid
sequenceDiagram
    participant IrqSaveGuard as "IrqSave Guard"
    participant ArchitectureLayer as "Architecture Layer"
    participant LoongArch64Implementation as "LoongArch64 Implementation"
    participant LoongArch64CSR as "LoongArch64 CSR"

    Note over IrqSaveGuard: Guard creation
    IrqSaveGuard ->> ArchitectureLayer: BaseGuard::acquire()
    ArchitectureLayer ->> LoongArch64Implementation: local_irq_save_and_disable()
    LoongArch64Implementation ->> LoongArch64CSR: csrxchg (clear IE)
    LoongArch64CSR -->> LoongArch64Implementation: Previous CSR value
    LoongArch64Implementation -->> ArchitectureLayer: Masked IE state
    ArchitectureLayer -->> IrqSaveGuard: Saved flags
    Note over IrqSaveGuard: Critical section execution
    Note over IrqSaveGuard: Guard destruction
    IrqSaveGuard ->> ArchitectureLayer: BaseGuard::release(flags)
    ArchitectureLayer ->> LoongArch64Implementation: local_irq_restore(flags)
    LoongArch64Implementation ->> LoongArch64CSR: csrxchg (restore IE)
    LoongArch64CSR -->> LoongArch64Implementation: Updated CSR
    LoongArch64Implementation -->> ArchitectureLayer: Return
    ArchitectureLayer -->> IrqSaveGuard: Return
```

Sources: [src/arch/loongarch64.rs(L6 - L17)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/arch/loongarch64.rs#L6-L17)