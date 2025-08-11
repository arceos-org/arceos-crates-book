# RISC-V Trap and Exception Handling

> **Relevant source files**
> * [src/aarch64/trap.S](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/trap.S)
> * [src/riscv/trap.S](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.S)
> * [src/riscv/trap.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs)

## Purpose and Scope

This page covers the RISC-V trap and exception handling implementation in axcpu, including both the assembly-level trap entry/exit mechanisms and the Rust-based trap dispatch logic. The system handles supervisor-mode and user-mode traps, including page faults, system calls, interrupts, and debugging exceptions.

For information about RISC-V context management and register state, see [4.1](/arceos-org/axcpu/4.1-risc-v-context-management). For system initialization and trap vector setup, see [4.3](/arceos-org/axcpu/4.3-risc-v-system-initialization). For cross-architecture trap handling abstractions, see [6.2](/arceos-org/axcpu/6.2-core-trap-handling-framework).

## Trap Handling Architecture

The RISC-V trap handling system operates in two phases: assembly-level trap entry/exit for performance-critical register save/restore operations, and Rust-based trap dispatch for high-level exception handling logic.

### Trap Flow Diagram

```mermaid
flowchart TD
HW["Hardware Exception/Interrupt"]
TV["trap_vector_base"]
SS["sscratch == 0?"]
STE[".Ltrap_entry_s(Supervisor trap)"]
UTE[".Ltrap_entry_u(User trap)"]
SRS["SAVE_REGS 0"]
URS["SAVE_REGS 1"]
RTH["riscv_trap_handler(tf, false)"]
RTH2["riscv_trap_handler(tf, true)"]
SC["scause analysis"]
PF["Page Faulthandle_page_fault()"]
BP["Breakpointhandle_breakpoint()"]
SYS["System Callhandle_syscall()"]
IRQ["Interrupthandle_trap!(IRQ)"]
SRR["RESTORE_REGS 0"]
URR["RESTORE_REGS 1"]
SRET["sret"]
SRET2["sret"]

BP --> SRR
HW --> TV
IRQ --> SRR
PF --> SRR
RTH --> SC
RTH2 --> SC
SC --> BP
SC --> IRQ
SC --> PF
SC --> SYS
SRR --> SRET
SRS --> RTH
SS --> STE
SS --> UTE
STE --> SRS
SYS --> URR
TV --> SS
URR --> SRET2
URS --> RTH2
UTE --> URS
```

Sources: [src/riscv/trap.S(L45 - L69)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.S#L45-L69) [src/riscv/trap.rs(L36 - L71)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L36-L71)

## Assembly Trap Entry Mechanism

The trap entry mechanism uses the `sscratch` register to distinguish between supervisor-mode and user-mode traps, implementing different register save/restore strategies for each privilege level.

### Register Save/Restore Strategy

|Trap Source|sscratch Value|Entry Point|Register Handling|
| --- | --- | --- | --- |
|Supervisor Mode|0|.Ltrap_entry_s|Basic register save, no privilege switch|
|User Mode|Non-zero (supervisor SP)|.Ltrap_entry_u|Full context switch including GP/TP registers|

### Trap Vector Implementation

```mermaid
flowchart TD
TV["trap_vector_base"]
SSW["csrrw sp, sscratch, sp"]
BZ["bnez sp, .Ltrap_entry_u"]
CSR["csrr sp, sscratch"]
UE[".Ltrap_entry_u"]
SE[".Ltrap_entry_s"]
SR0["SAVE_REGS 0Basic save"]
SR1["SAVE_REGS 1Context switch"]
CH["call riscv_trap_handler(tf, false)"]
CH2["call riscv_trap_handler(tf, true)"]

BZ --> CSR
BZ --> UE
CSR --> SE
SE --> SR0
SR0 --> CH
SR1 --> CH2
SSW --> BZ
TV --> SSW
UE --> SR1
```

Sources: [src/riscv/trap.S(L45 - L69)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.S#L45-L69)

### SAVE_REGS Macro Behavior

The `SAVE_REGS` macro implements different register handling strategies based on the trap source:

**Supervisor Mode (from_user=0)**:

* Saves all general-purpose registers to trap frame
* Preserves `sepc`, `sstatus`, and `sscratch`
* No privilege-level register switching

**User Mode (from_user=1)**:

* Saves user GP and TP registers to trap frame
* Loads supervisor GP and TP from trap frame offsets 2 and 3
* Switches to supervisor register context

Sources: [src/riscv/trap.S(L1 - L20)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.S#L1-L20)

## Rust Trap Handler Dispatch

The `riscv_trap_handler` function serves as the main dispatch point for all RISC-V traps, analyzing the `scause` register to determine trap type and invoking appropriate handlers.

### Trap Classification and Dispatch

```mermaid
flowchart TD
RTH["riscv_trap_handler(tf, from_user)"]
SCR["scause::read()"]
TTC["scause.cause().try_into::()"]
EXC["Exception Branch"]
INT["Interrupt Branch"]
UNK["Unknown trap"]
UEC["UserEnvCall(System Call)"]
LPF["LoadPageFault"]
SPF["StorePageFault"]
IPF["InstructionPageFault"]
BKP["Breakpoint"]
HIRQ["handle_trap!(IRQ)"]
HSC["handle_syscall(tf, tf.regs.a7)"]
HPF1["handle_page_fault(tf, READ, from_user)"]
HPF2["handle_page_fault(tf, WRITE, from_user)"]
HPF3["handle_page_fault(tf, EXECUTE, from_user)"]
HBK["handle_breakpoint(&mut tf.sepc)"]
PAN["panic!(Unknown trap)"]

BKP --> HBK
EXC --> BKP
EXC --> IPF
EXC --> LPF
EXC --> SPF
EXC --> UEC
INT --> HIRQ
IPF --> HPF3
LPF --> HPF1
RTH --> SCR
SCR --> TTC
SPF --> HPF2
TTC --> EXC
TTC --> INT
TTC --> UNK
UEC --> HSC
UNK --> PAN
```

Sources: [src/riscv/trap.rs(L36 - L71)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L36-L71)

## Specific Trap Type Handlers

### Page Fault Handling

The `handle_page_fault` function processes memory access violations by extracting the fault address from `stval` and determining access permissions:

```mermaid
flowchart TD
HPF["handle_page_fault(tf, access_flags, is_user)"]
USR["is_user?"]
UFLG["access_flags |= USER"]
VADDR["vaddr = va!(stval::read())"]
HTM["handle_trap!(PAGE_FAULT, vaddr, access_flags, is_user)"]
SUC["Success?"]
RET["Return"]
PAN["panic!(Unhandled Page Fault)"]

HPF --> USR
HTM --> SUC
SUC --> PAN
SUC --> RET
UFLG --> VADDR
USR --> UFLG
USR --> VADDR
VADDR --> HTM
```

**Page Fault Types**:

* `LoadPageFault`: Read access violation (`PageFaultFlags::READ`)
* `StorePageFault`: Write access violation (`PageFaultFlags::WRITE`)
* `InstructionPageFault`: Execute access violation (`PageFaultFlags::EXECUTE`)

Sources: [src/riscv/trap.rs(L19 - L34)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L19-L34) [src/riscv/trap.rs(L46 - L54)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L46-L54)

### System Call Handling

System calls are handled through the `UserEnvCall` exception when the `uspace` feature is enabled:

```
tf.regs.a0 = crate::trap::handle_syscall(tf, tf.regs.a7) as usize;
tf.sepc += 4;
```

The system call number is passed in register `a7`, and the return value is stored in `a0`. The program counter (`sepc`) is incremented by 4 to skip the `ecall` instruction.

Sources: [src/riscv/trap.rs(L42 - L45)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L42-L45)

### Breakpoint Handling

Breakpoint exceptions increment the program counter by 2 bytes to skip the compressed `ebreak` instruction:

```rust
fn handle_breakpoint(sepc: &mut usize) {
    debug!("Exception(Breakpoint) @ {sepc:#x} ");
    *sepc += 2
}
```

Sources: [src/riscv/trap.rs(L14 - L17)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L14-L17) [src/riscv/trap.rs(L55)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L55-L55)

### Interrupt Handling

Hardware interrupts are dispatched to the common trap handling framework using the `handle_trap!` macro with the `IRQ` trap type and `scause.bits()` as the interrupt number.

Sources: [src/riscv/trap.rs(L56 - L58)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L56-L58)

## Integration with Common Framework

The RISC-V trap handler integrates with axcpu's cross-architecture trap handling framework through several mechanisms:

### Trap Framework Integration

```mermaid
flowchart TD
subgraph subGraph1["Common Framework"]
    HTM["handle_trap! macro"]
    HSC["crate::trap::handle_syscall"]
    PFF["PageFaultFlags"]
end
subgraph subGraph0["RISC-V Specific"]
    RTH["riscv_trap_handler"]
    HPF["handle_page_fault"]
    HBP["handle_breakpoint"]
end

HPF --> HTM
HPF --> PFF
RTH --> HSC
RTH --> HTM
```

**Framework Components Used**:

* `PageFaultFlags`: Common page fault flag definitions
* `handle_trap!` macro: Architecture-agnostic trap dispatch
* `crate::trap::handle_syscall`: Common system call interface

Sources: [src/riscv/trap.rs(L1 - L6)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L1-L6) [src/riscv/trap.rs(L24)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L24-L24) [src/riscv/trap.rs(L43)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L43-L43) [src/riscv/trap.rs(L57)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L57-L57)

## Assembly Integration

The trap handling system integrates assembly code through the `global_asm!` macro, including architecture-specific macros and the trap frame size constant:

```javascript
core::arch::global_asm!(
    include_asm_macros!(),
    include_str!("trap.S"),
    trapframe_size = const core::mem::size_of::<TrapFrame>(),
);
```

This approach ensures type safety by using the Rust `TrapFrame` size directly in assembly code, preventing layout mismatches between Rust and assembly implementations.

Sources: [src/riscv/trap.rs(L8 - L12)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/trap.rs#L8-L12)