# x86_64 System Calls

> **Relevant source files**
> * [src/riscv/macros.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/riscv/macros.rs)
> * [src/x86_64/syscall.S](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/syscall.S)
> * [src/x86_64/syscall.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/syscall.rs)

This document covers the x86_64 system call implementation in the axcpu library, including the low-level assembly entry point, MSR configuration, and user-kernel transition mechanisms. The system call interface enables user space programs to request kernel services through the `syscall` instruction.

For general trap and exception handling beyond system calls, see [x86_64 Trap and Exception Handling](/arceos-org/axcpu/2.2-x86_64-trap-and-exception-handling). For user space context management across architectures, see [User Space Support](/arceos-org/axcpu/6.1-user-space-support).

## System Call Architecture Overview

The x86_64 system call implementation consists of three main components: MSR configuration during initialization, the assembly entry point that handles the low-level transition, and the Rust handler that processes the actual system call.

```mermaid
flowchart TD
subgraph subGraph2["Context Management"]
    SWAPGS["swapgs"]
    STACK_SWITCH["Stack Switch"]
    TRAPFRAME["TrapFrame Construction"]
    RESTORE["Context Restore"]
end
subgraph subGraph0["System Call Flow"]
    USER["User Space Process"]
    SYSCALL_INSTR["syscall instruction"]
    ENTRY["syscall_entry (Assembly)"]
    HANDLER["x86_syscall_handler (Rust)"]
    GENERIC["handle_syscall (Generic)"]
    SYSRET["sysretq instruction"]
end
subgraph Configuration["Configuration"]
    INIT["init_syscall()"]
    LSTAR["LSTAR MSR"]
    STAR["STAR MSR"]
    SFMASK["SFMASK MSR"]
    EFER["EFER MSR"]
end

ENTRY --> HANDLER
ENTRY --> STACK_SWITCH
ENTRY --> SWAPGS
ENTRY --> TRAPFRAME
GENERIC --> HANDLER
HANDLER --> GENERIC
HANDLER --> SYSRET
INIT --> EFER
INIT --> LSTAR
INIT --> SFMASK
INIT --> STAR
SYSCALL_INSTR --> ENTRY
SYSRET --> RESTORE
SYSRET --> USER
USER --> SYSCALL_INSTR
```

**Sources:** [src/x86_64/syscall.rs(L1 - L49)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/syscall.rs#L1-L49) [src/x86_64/syscall.S(L1 - L56)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/syscall.S#L1-L56)

## System Call Initialization

The `init_syscall` function configures the x86_64 Fast System Call mechanism by programming several Model Specific Registers (MSRs).

|MSR|Purpose|Configuration|
| --- | --- | --- |
|LSTAR|Long Syscall Target Address|Points tosyscall_entryassembly function|
|STAR|Syscall Target Address|Contains segment selectors for user/kernel code/data|
|SFMASK|Syscall Flag Mask|Masks specific RFLAGS during syscall execution|
|EFER|Extended Feature Enable|Enables System Call Extensions (SCE bit)|

```mermaid
flowchart TD
subgraph subGraph3["MSR Configuration in init_syscall()"]
    INIT_FUNC["init_syscall()"]
    subgraph subGraph2["Flag Masking"]
        TF["TRAP_FLAG"]
        IF["INTERRUPT_FLAG"]
        DF["DIRECTION_FLAG"]
        IOPL["IOPL_LOW | IOPL_HIGH"]
        AC["ALIGNMENT_CHECK"]
        NT["NESTED_TASK"]
    end
    subgraph subGraph1["GDT Selectors"]
        UCODE["UCODE64_SELECTOR"]
        UDATA["UDATA_SELECTOR"]
        KCODE["KCODE64_SELECTOR"]
        KDATA["KDATA_SELECTOR"]
    end
    subgraph subGraph0["MSR Setup"]
        LSTAR_SETUP["LStar::write(syscall_entry)"]
        STAR_SETUP["Star::write(selectors)"]
        SFMASK_SETUP["SFMask::write(flags)"]
        EFER_SETUP["Efer::update(SCE)"]
        KERNELGS_SETUP["KernelGsBase::write(0)"]
    end
end

INIT_FUNC --> EFER_SETUP
INIT_FUNC --> KERNELGS_SETUP
INIT_FUNC --> LSTAR_SETUP
INIT_FUNC --> SFMASK_SETUP
INIT_FUNC --> STAR_SETUP
SFMASK_SETUP --> AC
SFMASK_SETUP --> DF
SFMASK_SETUP --> IF
SFMASK_SETUP --> IOPL
SFMASK_SETUP --> NT
SFMASK_SETUP --> TF
STAR_SETUP --> KCODE
STAR_SETUP --> KDATA
STAR_SETUP --> UCODE
STAR_SETUP --> UDATA
```

The `SFMASK` register masks flags with value `0x47700`, ensuring that potentially dangerous flags like interrupts and trap flags are cleared during system call execution.

**Sources:** [src/x86_64/syscall.rs(L22 - L48)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/syscall.rs#L22-L48)

## Assembly Entry Point

The `syscall_entry` function in assembly handles the low-level transition from user space to kernel space. It performs critical operations including GS base swapping, stack switching, and trap frame construction.

```mermaid
sequenceDiagram
    participant UserSpace as "User Space"
    participant CPUHardware as "CPU Hardware"
    participant KernelSpace as "Kernel Space"

    UserSpace ->> CPUHardware: syscall instruction
    CPUHardware ->> KernelSpace: Jump to syscall_entry
    Note over KernelSpace: swapgs (switch to kernel GS)
    KernelSpace ->> KernelSpace: Save user RSP to percpu area
    KernelSpace ->> KernelSpace: Load kernel stack from TSS
    Note over KernelSpace: Build TrapFrame on stack
    KernelSpace ->> KernelSpace: Push user RSP, RFLAGS, RIP
    KernelSpace ->> KernelSpace: Push all general purpose registers
    KernelSpace ->> KernelSpace: Call x86_syscall_handler(trapframe)
    KernelSpace ->> KernelSpace: Pop all general purpose registers
    KernelSpace ->> KernelSpace: Restore user RSP, RFLAGS, RIP
    Note over KernelSpace: swapgs (switch to user GS)
    KernelSpace ->> CPUHardware: sysretq instruction
    CPUHardware ->> UserSpace: Return to user space
```

### Key Assembly Operations

The entry point performs these critical steps:

1. **GS Base Switching**: `swapgs` switches between user and kernel GS base registers
2. **Stack Management**: Saves user RSP and switches to kernel stack from TSS
3. **Register Preservation**: Pushes all general-purpose registers to build a `TrapFrame`
4. **Handler Invocation**: Calls `x86_syscall_handler` with the trap frame pointer
5. **Context Restoration**: Restores all registers and user context
6. **Return**: Uses `sysretq` to return to user space

**Sources:** [src/x86_64/syscall.S(L3 - L55)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/syscall.S#L3-L55)

## System Call Handler

The Rust-based system call handler processes the actual system call request after the assembly entry point has set up the execution context.

```mermaid
flowchart TD
subgraph subGraph2["Per-CPU Data"]
    USER_RSP["USER_RSP_OFFSET"]
    TSS_STACK["TSS privilege_stack_table"]
end
subgraph subGraph1["Data Flow"]
    TRAPFRAME_IN["TrapFrame (Input)"]
    SYSCALL_NUM["tf.rax (System Call Number)"]
    RESULT["Return Value"]
    TRAPFRAME_OUT["tf.rax (Updated)"]
end
subgraph subGraph0["Handler Flow"]
    ASM_ENTRY["syscall_entry (Assembly)"]
    RUST_HANDLER["x86_syscall_handler"]
    GENERIC_HANDLER["crate::trap::handle_syscall"]
    RETURN["Return to Assembly"]
end

ASM_ENTRY --> RUST_HANDLER
ASM_ENTRY --> TSS_STACK
ASM_ENTRY --> USER_RSP
GENERIC_HANDLER --> RESULT
GENERIC_HANDLER --> RUST_HANDLER
RESULT --> TRAPFRAME_OUT
RUST_HANDLER --> GENERIC_HANDLER
RUST_HANDLER --> RETURN
SYSCALL_NUM --> GENERIC_HANDLER
TRAPFRAME_IN --> RUST_HANDLER
```

The handler function is minimal but critical:

```rust
#[unsafe(no_mangle)]
pub(super) fn x86_syscall_handler(tf: &mut TrapFrame) {
    tf.rax = crate::trap::handle_syscall(tf, tf.rax as usize) as u64;
}
```

It extracts the system call number from `rax`, delegates to the generic system call handler, and stores the return value back in `rax` for return to user space.

**Sources:** [src/x86_64/syscall.rs(L17 - L20)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/syscall.rs#L17-L20)

## Per-CPU State Management

The system call implementation uses per-CPU variables to manage state during the user-kernel transition:

|Variable|Purpose|Usage|
| --- | --- | --- |
|USER_RSP_OFFSET|Stores user stack pointer|Saved during entry, restored during exit|
|TSS.privilege_stack_table|Kernel stack pointer|Loaded from TSS during stack switch|

```mermaid
flowchart TD
subgraph subGraph1["Assembly Operations"]
    SAVE_USER_RSP["mov gs:[USER_RSP_OFFSET], rsp"]
    LOAD_KERNEL_RSP["mov rsp, gs:[TSS + rsp0_offset]"]
    RESTORE_USER_RSP["mov rsp, [rsp - 2 * 8]"]
end
subgraph subGraph0["Per-CPU Memory Layout"]
    PERCPU_BASE["Per-CPU Base Address"]
    USER_RSP_VAR["USER_RSP_OFFSET"]
    TSS_VAR["TSS Structure"]
    TSS_RSP0["privilege_stack_table[0]"]
end

PERCPU_BASE --> TSS_VAR
PERCPU_BASE --> USER_RSP_VAR
SAVE_USER_RSP --> RESTORE_USER_RSP
TSS_RSP0 --> LOAD_KERNEL_RSP
TSS_VAR --> TSS_RSP0
USER_RSP_VAR --> SAVE_USER_RSP
```

The per-CPU storage ensures that system calls work correctly in multi-CPU environments where each CPU needs its own temporary storage for user state.

**Sources:** [src/x86_64/syscall.rs(L9 - L10)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/syscall.rs#L9-L10) [src/x86_64/syscall.S(L5 - L6)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/syscall.S#L5-L6) [src/x86_64/syscall.S(L52)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/syscall.S#L52-L52)

## Integration with Trap Framework

The x86_64 system call implementation integrates with the broader trap handling framework through the `handle_syscall` function, which provides architecture-independent system call processing.

```mermaid
flowchart TD
subgraph subGraph2["Return Path"]
    RESULT_VAL["Return Value"]
    RAX_UPDATE["tf.rax = result"]
    SYSRET["sysretq"]
end
subgraph subGraph1["Generic Trap Framework"]
    HANDLE_SYSCALL["crate::trap::handle_syscall"]
    SYSCALL_DISPATCH["System Call Dispatch"]
    SYSCALL_IMPL["Actual System Call Implementation"]
end
subgraph subGraph0["Architecture-Specific Layer"]
    SYSCALL_ENTRY["syscall_entry.S"]
    X86_HANDLER["x86_syscall_handler"]
    TRAPFRAME["TrapFrame"]
end

HANDLE_SYSCALL --> SYSCALL_DISPATCH
RAX_UPDATE --> SYSRET
RESULT_VAL --> RAX_UPDATE
SYSCALL_DISPATCH --> SYSCALL_IMPL
SYSCALL_ENTRY --> X86_HANDLER
SYSCALL_IMPL --> RESULT_VAL
TRAPFRAME --> HANDLE_SYSCALL
X86_HANDLER --> TRAPFRAME
```

This layered approach allows the generic trap framework to handle system call logic while the x86_64-specific code manages the low-level hardware interface and calling conventions.

**Sources:** [src/x86_64/syscall.rs(L18 - L19)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/syscall.rs#L18-L19)