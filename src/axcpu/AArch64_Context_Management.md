# AArch64 Context Management

> **Relevant source files**
> * [src/aarch64/context.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/context.rs)

This document covers CPU context management for the AArch64 architecture within the axcpu library. It details the data structures, mechanisms, and assembly implementations used for task switching, exception handling, and state preservation on ARM64 systems.

The scope includes the `TaskContext` structure for task switching, `TrapFrame` for exception handling, and `FpState` for floating-point operations. For information about how these contexts are used during exception processing, see [AArch64 Trap and Exception Handling](/arceos-org/axcpu/3.2-aarch64-trap-and-exception-handling). For system initialization and setup procedures, see [AArch64 System Initialization](/arceos-org/axcpu/3.3-aarch64-system-initialization).

## Context Data Structures

The AArch64 context management revolves around three primary data structures, each serving different aspects of CPU state management.

### Context Structure Overview

```mermaid
flowchart TD
subgraph subGraph3["FpState Fields"]
    FP_REGS["regs[32]: u128V0-V31 SIMD Registers"]
    FP_CTRL["fpcr/fpsr: u32Control/Status Registers"]
end
subgraph subGraph2["TrapFrame Fields"]
    TF_REGS["r[31]: u64General Purpose Registers"]
    TF_USP["usp: u64User Stack Pointer"]
    TF_ELR["elr: u64Exception Link Register"]
    TF_SPSR["spsr: u64Saved Process Status"]
end
subgraph subGraph1["TaskContext Fields"]
    TC_SP["sp: u64Stack Pointer"]
    TC_TPIDR["tpidr_el0: u64Thread Local Storage"]
    TC_REGS["r19-r29, lr: u64Callee-saved Registers"]
    TC_TTBR["ttbr0_el1: PhysAddrPage Table Root"]
    TC_FP["fp_state: FpStateFP/SIMD State"]
end
subgraph subGraph0["AArch64 Context Management"]
    TaskContext["TaskContextTask Switching Context"]
    TrapFrame["TrapFrameException Context"]
    FpState["FpStateFP/SIMD Context"]
end

FpState --> FP_CTRL
FpState --> FP_REGS
TC_FP --> FpState
TaskContext --> TC_FP
TaskContext --> TC_REGS
TaskContext --> TC_SP
TaskContext --> TC_TPIDR
TaskContext --> TC_TTBR
TrapFrame --> TF_ELR
TrapFrame --> TF_REGS
TrapFrame --> TF_SPSR
TrapFrame --> TF_USP
```

Sources: [src/aarch64/context.rs(L8 - L124)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/context.rs#L8-L124)

### TaskContext Structure

The `TaskContext` structure maintains the minimal set of registers needed for task switching, focusing on callee-saved registers and system state.

|Field|Type|Purpose|
| --- | --- | --- |
|sp|u64|Stack pointer register|
|tpidr_el0|u64|Thread pointer for TLS|
|r19-r29|u64|Callee-saved general registers|
|lr|u64|Link register (r30)|
|ttbr0_el1|PhysAddr|User page table root (uspace feature)|
|fp_state|FpState|Floating-point state (fp-simd feature)|

The structure is conditionally compiled based on enabled features, with `ttbr0_el1` only included with the `uspace` feature and `fp_state` only with the `fp-simd` feature.

Sources: [src/aarch64/context.rs(L104 - L124)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/context.rs#L104-L124)

### TrapFrame Structure

The `TrapFrame` captures the complete CPU state during exceptions and system calls, preserving all general-purpose registers and processor state.

```mermaid
flowchart TD
subgraph subGraph0["TrapFrame Memory Layout"]
    R0_30["r[0-30]General Registers248 bytes"]
    USP["uspUser Stack Pointer8 bytes"]
    ELR["elrException Link Register8 bytes"]
    SPSR["spsrSaved Process Status8 bytes"]
end

ELR --> SPSR
USP --> ELR
```

The `TrapFrame` provides accessor methods for system call arguments through `arg0()` through `arg5()` methods, which map to registers `r[0]` through `r[5]` respectively.

Sources: [src/aarch64/context.rs(L8 - L63)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/context.rs#L8-L63)

### FpState Structure

The `FpState` structure manages floating-point and SIMD register state, with 16-byte alignment required for proper SIMD operations.

```mermaid
flowchart TD
subgraph Operations["Operations"]
    SAVE["save()CPU → Memory"]
    RESTORE["restore()Memory → CPU"]
end
subgraph subGraph0["FpState Structure"]
    REGS["regs[32]: u128V0-V31 SIMD/FP Registers512 bytes total"]
    FPCR["fpcr: u32Floating-Point Control Register"]
    FPSR["fpsr: u32Floating-Point Status Register"]
end

FPCR --> SAVE
FPSR --> SAVE
REGS --> SAVE
RESTORE --> FPCR
RESTORE --> FPSR
RESTORE --> REGS
SAVE --> RESTORE
```

Sources: [src/aarch64/context.rs(L66 - L88)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/context.rs#L66-L88)

## Context Switching Mechanism

Task switching on AArch64 involves saving the current task's callee-saved registers and restoring the next task's context, with optional handling of floating-point state and page tables.

### Context Switch Flow

```mermaid
sequenceDiagram
    participant CurrentTaskContext as "Current TaskContext"
    participant NextTaskContext as "Next TaskContext"
    participant AArch64CPU as "AArch64 CPU"
    participant MemoryManagementUnit as "Memory Management Unit"

    CurrentTaskContext ->> CurrentTaskContext: Check fp-simd feature
    alt fp-simd enabled
        CurrentTaskContext ->> AArch64CPU: Save FP/SIMD state
        NextTaskContext ->> AArch64CPU: Restore FP/SIMD state
    end
    CurrentTaskContext ->> CurrentTaskContext: Check uspace feature
    alt uspace enabled and ttbr0_el1 differs
        NextTaskContext ->> MemoryManagementUnit: Update page table root
        MemoryManagementUnit ->> MemoryManagementUnit: Flush TLB
    end
    CurrentTaskContext ->> AArch64CPU: context_switch(current, next)
    Note over AArch64CPU: Assembly context switch
    AArch64CPU ->> CurrentTaskContext: Save callee-saved registers
    AArch64CPU ->> AArch64CPU: Restore next task registers
    AArch64CPU ->> NextTaskContext: Load context to CPU
```

The `switch_to` method orchestrates the complete context switch:

1. **FP/SIMD State**: If `fp-simd` feature is enabled, saves current FP state and restores next task's FP state
2. **Page Table Switching**: If `uspace` feature is enabled and page tables differ, updates `ttbr0_el1` and flushes TLB
3. **Register Context**: Calls assembly `context_switch` function to handle callee-saved registers

Sources: [src/aarch64/context.rs(L160 - L172)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/context.rs#L160-L172)

### Assembly Context Switch Implementation

The low-level context switching is implemented in naked assembly within the `context_switch` function:

```mermaid
flowchart TD
subgraph subGraph0["context_switch Assembly Flow"]
    SAVE_START["Save Current Contextstp x29,x30 [x0,12*8]"]
    SAVE_REGS["Save Callee-Savedr19-r28 to memory"]
    SAVE_SP_TLS["Save SP and TLSmov x19,sp; mrs x20,tpidr_el0"]
    RESTORE_SP_TLS["Restore SP and TLSmov sp,x19; msr tpidr_el0,x20"]
    RESTORE_REGS["Restore Callee-Savedr19-r28 from memory"]
    RESTORE_END["Restore Frameldp x29,x30 [x1,12*8]"]
    RET["Returnret"]
end

RESTORE_END --> RET
RESTORE_REGS --> RESTORE_END
RESTORE_SP_TLS --> RESTORE_REGS
SAVE_REGS --> SAVE_SP_TLS
SAVE_SP_TLS --> RESTORE_SP_TLS
SAVE_START --> SAVE_REGS
```

The assembly implementation uses paired load/store instructions (`stp`/`ldp`) for efficiency, handling registers in pairs and preserving the AArch64 calling convention.

Sources: [src/aarch64/context.rs(L175 - L203)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/context.rs#L175-L203)

## Feature-Conditional Context Management

The AArch64 context management includes several optional features that extend the base functionality.

### Feature Integration Overview

```mermaid
flowchart TD
subgraph subGraph2["Feature Operations"]
    SET_PT["set_page_table_root()Update ttbr0_el1"]
    SWITCH_PT["Page table switchingin switch_to()"]
    FP_SAVE["fp_state.save()CPU → Memory"]
    FP_RESTORE["fp_state.restore()Memory → CPU"]
    TLS_INIT["TLS initializationin init()"]
end
subgraph subGraph1["Optional Features"]
    USPACE["uspace Featurettbr0_el1: PhysAddr"]
    FPSIMD["fp-simd Featurefp_state: FpState"]
    TLS["tls Supporttpidr_el0 management"]
end
subgraph subGraph0["Base Context"]
    BASE["TaskContext Basesp, tpidr_el0, r19-r29, lr"]
end

BASE --> FPSIMD
BASE --> TLS
BASE --> USPACE
FPSIMD --> FP_RESTORE
FPSIMD --> FP_SAVE
TLS --> TLS_INIT
USPACE --> SET_PT
USPACE --> SWITCH_PT
```

### User Space Support

When the `uspace` feature is enabled, `TaskContext` includes the `ttbr0_el1` field for managing user-space page tables. The `set_page_table_root` method allows updating the page table root, and context switching automatically handles page table updates and TLB flushes when switching between tasks with different address spaces.

### Floating-Point State Management

The `fp-simd` feature enables comprehensive floating-point and SIMD state management through the `FpState` structure. The assembly implementations `fpstate_save` and `fpstate_restore` handle all 32 vector registers (V0-V31) plus control registers using quad-word load/store instructions.

Sources: [src/aarch64/context.rs(L77 - L88)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/context.rs#L77-L88) [src/aarch64/context.rs(L120 - L123)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/context.rs#L120-L123) [src/aarch64/context.rs(L147 - L154)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/context.rs#L147-L154) [src/aarch64/context.rs(L206 - L267)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/context.rs#L206-L267)

## Task Initialization

New tasks are initialized through the `TaskContext::init` method, which sets up the minimal execution environment:

```mermaid
flowchart TD
subgraph subGraph1["TaskContext Setup"]
    SET_SP["self.sp = kstack_top"]
    SET_LR["self.lr = entry"]
    SET_TLS["self.tpidr_el0 = tls_area"]
end
subgraph subGraph0["Task Initialization Parameters"]
    ENTRY["entry: usizeTask entry point"]
    KSTACK["kstack_top: VirtAddrKernel stack top"]
    TLS_AREA["tls_area: VirtAddrThread-local storage"]
end

ENTRY --> SET_LR
KSTACK --> SET_SP
TLS_AREA --> SET_TLS
```

The initialization sets the stack pointer to the kernel stack top, the link register to the task entry point, and the thread pointer for thread-local storage support.

Sources: [src/aarch64/context.rs(L140 - L145)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/context.rs#L140-L145)