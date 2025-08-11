# x86_64 Context Management

> **Relevant source files**
> * [src/x86_64/context.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/context.rs)

This document covers the x86_64 CPU context management implementation in axcpu, focusing on the core data structures and mechanisms used for task context switching, exception handling, and state preservation. The implementation provides both kernel-level task switching and user space context management capabilities.

For x86_64 trap and exception handling mechanisms, see [x86_64 Trap and Exception Handling](/arceos-org/axcpu/2.2-x86_64-trap-and-exception-handling). For system call implementation details, see [x86_64 System Calls](/arceos-org/axcpu/2.3-x86_64-system-calls).

## Core Context Data Structures

The x86_64 context management system uses several key data structures to manage CPU state across different scenarios:

```mermaid
flowchart TD
subgraph subGraph2["Hardware Integration"]
    CPU["x86_64 CPU"]
    Stack["Kernel Stack"]
    Registers["CPU Registers"]
end
subgraph subGraph0["Core Context Types"]
    TaskContext["TaskContextMain task switching context"]
    TrapFrame["TrapFrameException/interrupt context"]
    ContextSwitchFrame["ContextSwitchFrameInternal context switch frame"]
    ExtendedState["ExtendedStateFP/SIMD state container"]
    FxsaveArea["FxsaveArea512-byte FXSAVE region"]
end

ContextSwitchFrame --> Registers
ExtendedState --> CPU
ExtendedState --> FxsaveArea
TaskContext --> ContextSwitchFrame
TaskContext --> ExtendedState
TaskContext --> Stack
TrapFrame --> Registers
```

Sources: [src/x86_64/context.rs(L1 - L291)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/context.rs#L1-L291)

### TaskContext Structure

The `TaskContext` struct represents the complete saved hardware state of a task, containing all necessary information for context switching:

|Field|Type|Purpose|Feature Gate|
| --- | --- | --- | --- |
|kstack_top|VirtAddr|Top of kernel stack|Always|
|rsp|u64|Stack pointer after register saves|Always|
|fs_base|usize|Thread Local Storage base|Always|
|gs_base|usize|User space GS base register|uspace|
|ext_state|ExtendedState|FP/SIMD state|fp-simd|
|cr3|PhysAddr|Page table root|uspace|

Sources: [src/x86_64/context.rs(L166 - L183)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/context.rs#L166-L183)

### TrapFrame Structure

The `TrapFrame` captures the complete CPU register state when a trap (interrupt or exception) occurs, containing all general-purpose registers plus trap-specific information:

```mermaid
flowchart TD
subgraph subGraph1["Syscall Interface"]
    ARG0["arg0() -> rdi"]
    ARG1["arg1() -> rsi"]
    ARG2["arg2() -> rdx"]
    ARG3["arg3() -> r10"]
    ARG4["arg4() -> r8"]
    ARG5["arg5() -> r9"]
    USER_CHECK["is_user() -> cs & 3 == 3"]
end
subgraph subGraph0["TrapFrame Layout"]
    GPR["General Purpose Registersrax, rbx, rcx, rdx, rbp, rsi, rdir8, r9, r10, r11, r12, r13, r14, r15"]
    TRAP_INFO["Trap Informationvector, error_code"]
    CPU_PUSHED["CPU-Pushed Fieldsrip, cs, rflags, rsp, ss"]
end

CPU_PUSHED --> USER_CHECK
GPR --> ARG0
GPR --> ARG1
GPR --> ARG2
GPR --> ARG3
GPR --> ARG4
GPR --> ARG5
```

Sources: [src/x86_64/context.rs(L4 - L72)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/context.rs#L4-L72)

## Context Switching Mechanism

The context switching process involves saving the current task's state and restoring the next task's state through a coordinated sequence of operations:

```mermaid
flowchart TD
subgraph subGraph1["Feature Gates"]
    FP_GATE["fp-simd feature"]
    TLS_GATE["tls feature"]
    USPACE_GATE["uspace feature"]
end
subgraph subGraph0["Context Switch Flow"]
    START["switch_to() called"]
    SAVE_FP["Save FP/SIMD stateext_state.save()"]
    RESTORE_FP["Restore FP/SIMD statenext_ctx.ext_state.restore()"]
    SAVE_TLS["Save current TLSread_thread_pointer()"]
    RESTORE_TLS["Restore next TLSwrite_thread_pointer()"]
    SAVE_GS["Save GS baserdmsr(IA32_KERNEL_GSBASE)"]
    RESTORE_GS["Restore GS basewrmsr(IA32_KERNEL_GSBASE)"]
    UPDATE_TSS["Update TSS RSP0write_tss_rsp0()"]
    SWITCH_PT["Switch page tablewrite_user_page_table()"]
    ASM_SWITCH["Assembly context switchcontext_switch()"]
    END["Context switch complete"]
end

ASM_SWITCH --> END
RESTORE_FP --> FP_GATE
RESTORE_FP --> SAVE_TLS
RESTORE_GS --> UPDATE_TSS
RESTORE_GS --> USPACE_GATE
RESTORE_TLS --> SAVE_GS
RESTORE_TLS --> TLS_GATE
SAVE_FP --> FP_GATE
SAVE_FP --> RESTORE_FP
SAVE_GS --> RESTORE_GS
SAVE_GS --> USPACE_GATE
SAVE_TLS --> RESTORE_TLS
SAVE_TLS --> TLS_GATE
START --> SAVE_FP
SWITCH_PT --> ASM_SWITCH
SWITCH_PT --> USPACE_GATE
UPDATE_TSS --> SWITCH_PT
UPDATE_TSS --> USPACE_GATE
```

Sources: [src/x86_64/context.rs(L242 - L265)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/context.rs#L242-L265)

### Assembly Context Switch Implementation

The low-level context switching is implemented in assembly using a naked function that saves and restores callee-saved registers:

```mermaid
flowchart TD
subgraph subGraph0["context_switch Assembly"]
    PUSH["Push callee-saved registersrbp, rbx, r12-r15"]
    SAVE_RSP["Save current RSPmov [rdi], rsp"]
    LOAD_RSP["Load next RSPmov rsp, [rsi]"]
    POP["Pop callee-saved registersr15-r12, rbx, rbp"]
    RET["Return to new taskret"]
end

LOAD_RSP --> POP
POP --> RET
PUSH --> SAVE_RSP
SAVE_RSP --> LOAD_RSP
```

Sources: [src/x86_64/context.rs(L268 - L290)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/context.rs#L268-L290)

## Extended State Management

The x86_64 architecture provides extensive floating-point and SIMD capabilities that require specialized state management:

### FxsaveArea Structure

The `FxsaveArea` represents the 512-byte memory region used by the FXSAVE/FXRSTOR instructions:

|Field|Type|Purpose|
| --- | --- | --- |
|fcw|u16|FPU Control Word|
|fsw|u16|FPU Status Word|
|ftw|u16|FPU Tag Word|
|fop|u16|FPU Opcode|
|fip|u64|FPU Instruction Pointer|
|fdp|u64|FPU Data Pointer|
|mxcsr|u32|MXCSR Register|
|mxcsr_mask|u32|MXCSR Mask|
|st|[u64; 16]|ST0-ST7 FPU registers|
|xmm|[u64; 32]|XMM0-XMM15 registers|

Sources: [src/x86_64/context.rs(L86 - L107)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/context.rs#L86-L107)

### ExtendedState Operations

The `ExtendedState` provides methods for saving and restoring FP/SIMD state:

```mermaid
flowchart TD
subgraph subGraph2["Hardware Interface"]
    CPU_FPU["CPU FPU/SIMD Units"]
    FXSAVE_AREA["512-byte aligned memory"]
end
subgraph subGraph1["Default Values"]
    FCW["fcw = 0x37fStandard FPU control"]
    FTW["ftw = 0xffffAll tags empty"]
    MXCSR["mxcsr = 0x1f80Standard SIMD control"]
end
subgraph subGraph0["ExtendedState Operations"]
    SAVE["save()_fxsave64()"]
    RESTORE["restore()_fxrstor64()"]
    DEFAULT["default()Initialize with standard values"]
end

DEFAULT --> FCW
DEFAULT --> FTW
DEFAULT --> MXCSR
RESTORE --> CPU_FPU
RESTORE --> FXSAVE_AREA
SAVE --> CPU_FPU
SAVE --> FXSAVE_AREA
```

Sources: [src/x86_64/context.rs(L115 - L137)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/context.rs#L115-L137)

## Task Context Initialization

The task context initialization process sets up a new task for execution:

```mermaid
flowchart TD
subgraph subGraph2["Stack Layout"]
    STACK_TOP["Stack top (16-byte aligned)"]
    PADDING["8-byte padding"]
    SWITCH_FRAME["ContextSwitchFramerip = entry point"]
end
subgraph subGraph1["Default Values"]
    KSTACK["kstack_top = 0"]
    RSP["rsp = 0"]
    FS_BASE["fs_base = 0"]
    CR3["cr3 = kernel page table"]
    EXT_STATE["ext_state = default"]
    GS_BASE["gs_base = 0"]
end
subgraph subGraph0["Task Initialization"]
    NEW["new()Create empty context"]
    INIT["init()Setup for execution"]
    SETUP_STACK["Setup kernel stack16-byte alignment"]
    CREATE_FRAME["Create ContextSwitchFrameSet entry point"]
    SET_TLS["Set TLS basefs_base = tls_area"]
end

CREATE_FRAME --> SET_TLS
INIT --> SETUP_STACK
NEW --> CR3
NEW --> EXT_STATE
NEW --> FS_BASE
NEW --> GS_BASE
NEW --> KSTACK
NEW --> RSP
PADDING --> SWITCH_FRAME
SETUP_STACK --> CREATE_FRAME
SETUP_STACK --> STACK_TOP
STACK_TOP --> PADDING
```

Sources: [src/x86_64/context.rs(L185 - L227)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/context.rs#L185-L227)

## User Space Context Management

When the `uspace` feature is enabled, the context management system provides additional support for user space processes:

### User Space Fields

|Field|Purpose|Usage|
| --- | --- | --- |
|gs_base|User space GS base register|Saved/restored via MSR operations|
|cr3|Page table root|Updated during context switch|

### Page Table Management

The context switching process includes page table switching when transitioning between user space tasks:

```mermaid
flowchart TD
subgraph subGraph1["TSS Update"]
    UPDATE_TSS["write_tss_rsp0(next_ctx.kstack_top)"]
    KERNEL_STACK["Update kernel stack for syscalls"]
end
subgraph subGraph0["Page Table Switch Logic"]
    CHECK["Compare next_ctx.cr3 with current cr3"]
    SWITCH["write_user_page_table(next_ctx.cr3)"]
    FLUSH["TLB automatically flushed"]
    SKIP["Skip page table switch"]
end

CHECK --> SKIP
CHECK --> SWITCH
SWITCH --> FLUSH
SWITCH --> UPDATE_TSS
UPDATE_TSS --> KERNEL_STACK
```

Sources: [src/x86_64/context.rs(L253 - L263)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/context.rs#L253-L263)

## System Call Argument Extraction

The `TrapFrame` provides convenient methods for extracting system call arguments following the x86_64 calling convention:

|Method|Register|Purpose|
| --- | --- | --- |
|arg0()|rdi|First argument|
|arg1()|rsi|Second argument|
|arg2()|rdx|Third argument|
|arg3()|r10|Fourth argument (note: r10, not rcx)|
|arg4()|r8|Fifth argument|
|arg5()|r9|Sixth argument|
|is_user()|cs & 3|Check if trap originated from user space|

Sources: [src/x86_64/context.rs(L37 - L71)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/context.rs#L37-L71)