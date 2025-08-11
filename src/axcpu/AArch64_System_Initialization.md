# AArch64 System Initialization

> **Relevant source files**
> * [src/aarch64/init.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/init.rs)

This document covers the AArch64 system initialization procedures provided by the axcpu library. The initialization process includes exception level transitions, Memory Management Unit (MMU) configuration, and trap handler setup. These functions are primarily used during system bootstrap to establish a proper execution environment at EL1.

For detailed information about AArch64 trap and exception handling mechanisms, see [AArch64 Trap and Exception Handling](/arceos-org/axcpu/3.2-aarch64-trap-and-exception-handling). For AArch64 context management including TaskContext and TrapFrame structures, see [AArch64 Context Management](/arceos-org/axcpu/3.1-aarch64-context-management).

## Exception Level Transition

AArch64 systems support multiple exception levels (EL0-EL3), with EL3 being the highest privilege level and EL0 being user space. The `switch_to_el1()` function handles the transition from higher exception levels (EL2 or EL3) down to EL1, which is the typical kernel execution level.

### Exception Level Transition Flow

```mermaid
flowchart TD
START["switch_to_el1()"]
CHECK_EL["CurrentEL.read(CurrentEL::EL)"]
EL3_CHECK["current_el == 3?"]
EL2_PLUS["current_el >= 2?"]
EL3_CONFIG["Configure EL3→EL2 transitionSCR_EL3.write()SPSR_EL3.write()ELR_EL3.set()"]
EL2_CONFIG["Configure EL2→EL1 transitionCNTHCTL_EL2.modify()CNTVOFF_EL2.set()HCR_EL2.write()SPSR_EL2.write()SP_EL1.set()ELR_EL2.set()"]
ERET["aarch64_cpu::asm::eret()"]
DONE["Return (already in EL1)"]

CHECK_EL --> EL2_PLUS
EL2_CONFIG --> ERET
EL2_PLUS --> DONE
EL2_PLUS --> EL3_CHECK
EL3_CHECK --> EL2_CONFIG
EL3_CHECK --> EL3_CONFIG
EL3_CONFIG --> EL2_CONFIG
ERET --> DONE
START --> CHECK_EL
```

Sources: [src/aarch64/init.rs(L15 - L52)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/init.rs#L15-L52)

The transition process configures multiple system registers:

|Register|Purpose|Configuration|
| --- | --- | --- |
|SCR_EL3|Secure Configuration Register|Non-secure world, HVC enabled, AArch64|
|SPSR_EL3/EL2|Saved Program Status Register|EL1h mode, interrupts masked|
|ELR_EL3/EL2|Exception Link Register|Return address from LR|
|HCR_EL2|Hypervisor Configuration Register|EL1 in AArch64 mode|
|CNTHCTL_EL2|Counter-timer Hypervisor Control|Enable EL1 timer access|

## MMU Initialization

The Memory Management Unit initialization is handled by the `init_mmu()` function, which configures address translation and enables caching. This function sets up a dual page table configuration using both TTBR0_EL1 and TTBR1_EL1.

### MMU Configuration Sequence

```mermaid
flowchart TD
INIT_MMU["init_mmu(root_paddr)"]
SET_MAIR["MAIR_EL1.set(MemAttr::MAIR_VALUE)"]
CONFIG_TCR["Configure TCR_EL1- Page size: 4KB- VA size: 48 bits- PA size: 48 bits"]
SET_TTBR["Set page table baseTTBR0_EL1.set(root_paddr)TTBR1_EL1.set(root_paddr)"]
FLUSH_TLB["crate::asm::flush_tlb(None)"]
ENABLE_MMU["SCTLR_EL1.modify()Enable MMU + I-cache + D-cache"]
ISB["barrier::isb(barrier::SY)"]

CONFIG_TCR --> ISB
ENABLE_MMU --> ISB
FLUSH_TLB --> ENABLE_MMU
INIT_MMU --> SET_MAIR
ISB --> SET_TTBR
SET_MAIR --> CONFIG_TCR
SET_TTBR --> FLUSH_TLB
```

Sources: [src/aarch64/init.rs(L54 - L95)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/init.rs#L54-L95)

### Memory Attribute Configuration

The MMU initialization configures memory attributes through several system registers:

|Register|Configuration|Purpose|
| --- | --- | --- |
|MAIR_EL1|Memory attribute encoding|Defines memory types (Normal, Device, etc.)|
|TCR_EL1|Translation control|Page size, address ranges, cacheability|
|TTBR0_EL1|Translation table base 0|User space page table root|
|TTBR1_EL1|Translation table base 1|Kernel space page table root|
|SCTLR_EL1|System control|MMU enable, cache enable|

The function configures both translation table base registers to use the same physical address, enabling a unified address space configuration at initialization time.

## Trap Handler Initialization

The `init_trap()` function sets up the exception handling infrastructure by configuring the exception vector table base address and initializing user space memory protection.

### Trap Initialization Components

```mermaid
flowchart TD
subgraph subGraph1["User Space Protection"]
    TTBR0_ZERO["TTBR0_EL1 = 0Blocks low address access"]
    USER_ISOLATION["User space isolationPrevents unauthorized access"]
end
subgraph subGraph0["Exception Vector Setup"]
    VBAR_EL1["VBAR_EL1 registerPoints to exception_vector_base"]
    VECTOR_TABLE["Exception vector tableSynchronous, IRQ, FIQ, SError"]
end
INIT_TRAP["init_trap()"]
GET_VECTOR["extern exception_vector_base()"]
SET_VBAR["crate::asm::write_exception_vector_base()"]
CLEAR_TTBR0["crate::asm::write_user_page_table(0)"]

CLEAR_TTBR0 --> TTBR0_ZERO
GET_VECTOR --> SET_VBAR
INIT_TRAP --> GET_VECTOR
SET_VBAR --> CLEAR_TTBR0
SET_VBAR --> VBAR_EL1
TTBR0_ZERO --> USER_ISOLATION
VBAR_EL1 --> VECTOR_TABLE
```

Sources: [src/aarch64/init.rs(L97 - L109)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/init.rs#L97-L109)

The function performs two critical operations:

1. **Vector Base Setup**: Points `VBAR_EL1` to the `exception_vector_base` symbol, which contains the exception handlers
2. **User Space Protection**: Sets `TTBR0_EL1` to 0, effectively disabling user space address translation until a proper user page table is loaded

## System Initialization Sequence

The complete AArch64 system initialization follows a specific sequence to ensure proper system state transitions and hardware configuration.

### Complete Initialization Flow

```mermaid
flowchart TD
subgraph subGraph2["Exception Handling Setup"]
    SET_VBAR_OP["Set VBAR_EL1 to exception_vector_base"]
    PROTECT_USER["Zero TTBR0_EL1 for protection"]
end
subgraph subGraph1["Memory Management Setup"]
    MMU_REGS["Configure MMU registersMAIR_EL1, TCR_EL1, TTBR0/1_EL1"]
    ENABLE_MMU_OP["Enable SCTLR_EL1.M + C + I"]
end
subgraph subGraph0["Exception Level Transition"]
    EL_REGS["Configure EL transition registersSCR_EL3, SPSR_EL3, HCR_EL2, etc."]
    ERET_OP["Execute ERET instruction"]
end
BOOT["System Boot(EL2/EL3)"]
SWITCH["switch_to_el1()"]
EL1_MODE["Running in EL1"]
INIT_MMU_STEP["init_mmu(root_paddr)"]
MMU_ENABLED["MMU + Caches Enabled"]
INIT_TRAP_STEP["init_trap()"]
SYSTEM_READY["System Ready"]

BOOT --> SWITCH
EL1_MODE --> INIT_MMU_STEP
EL_REGS --> ERET_OP
ENABLE_MMU_OP --> MMU_ENABLED
ERET_OP --> EL1_MODE
INIT_MMU_STEP --> MMU_REGS
INIT_TRAP_STEP --> SET_VBAR_OP
MMU_ENABLED --> INIT_TRAP_STEP
MMU_REGS --> ENABLE_MMU_OP
PROTECT_USER --> SYSTEM_READY
SET_VBAR_OP --> PROTECT_USER
SWITCH --> EL_REGS
```

Sources: [src/aarch64/init.rs(L1 - L110)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/init.rs#L1-L110)

### Safety Considerations

All initialization functions are marked as `unsafe` due to their privileged operations:

* **`switch_to_el1()`**: Changes CPU execution mode and privilege level
* **`init_mmu()`**: Modifies address translation configuration, which can affect memory access behavior
* **Assembly operations**: Direct register manipulation and barrier instructions require careful sequencing

The initialization sequence must be performed in the correct order, as MMU configuration depends on being in the proper exception level, and trap handling setup assumes MMU functionality is available.