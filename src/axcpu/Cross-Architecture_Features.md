# Cross-Architecture Features

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/lib.rs)
> * [src/trap.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/trap.rs)

## Purpose and Scope

This document covers features and abstractions in the axcpu library that work uniformly across all supported architectures (x86_64, AArch64, RISC-V, and LoongArch64). These cross-architecture features provide unified interfaces and common functionality that abstract away architecture-specific implementation details.

For architecture-specific implementations, see the dedicated architecture pages: [x86_64 Architecture](/arceos-org/axcpu/2-x86_64-architecture), [AArch64 Architecture](/arceos-org/axcpu/3-aarch64-architecture), [RISC-V Architecture](/arceos-org/axcpu/4-risc-v-architecture), and [LoongArch64 Architecture](/arceos-org/axcpu/5-loongarch64-architecture). For user space specific functionality, see [User Space Support](/arceos-org/axcpu/6.1-user-space-support).

## Unified Trap Handling Framework

The axcpu library provides a distributed trap handling system that allows external code to register handlers for various types of traps and exceptions. This system works consistently across all supported architectures through the `linkme` crate's distributed slice mechanism.

### Trap Handler Registration

The core trap handling framework defines three types of distributed handler slices:

```mermaid
flowchart TD
subgraph subGraph2["Handler Invocation"]
    HANDLE_TRAP["handle_trap! macroDispatches to first handler"]
    HANDLE_SYSCALL["handle_syscall functionDirect syscall dispatch"]
end
subgraph subGraph1["Registration Mechanism"]
    LINKME["linkme::distributed_sliceCompile-time collection"]
    DEF_TRAP["def_trap_handler macroCreates handler slices"]
    REG_TRAP["register_trap_handler macroRegisters individual handlers"]
end
subgraph subGraph0["Trap Handler Types"]
    IRQ["IRQ[fn(usize) -> bool]Hardware interrupts"]
    PAGE_FAULT["PAGE_FAULT[fn(VirtAddr, PageFaultFlags, bool) -> bool]Memory access violations"]
    SYSCALL["SYSCALL[fn(&TrapFrame, usize) -> isize]System calls (uspace feature)"]
end

DEF_TRAP --> IRQ
DEF_TRAP --> PAGE_FAULT
DEF_TRAP --> SYSCALL
IRQ --> HANDLE_TRAP
LINKME --> DEF_TRAP
PAGE_FAULT --> HANDLE_TRAP
REG_TRAP --> IRQ
REG_TRAP --> PAGE_FAULT
REG_TRAP --> SYSCALL
SYSCALL --> HANDLE_SYSCALL
```

**Sources:** [src/trap.rs(L10 - L22)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/trap.rs#L10-L22) [src/trap.rs(L6 - L7)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/trap.rs#L6-L7)

### Handler Dispatch Mechanism

The trap handling system uses a macro-based dispatch mechanism that supports single-handler registration. When multiple handlers are registered for the same trap type, the system issues a warning and uses only the first handler.

```mermaid
flowchart TD
subgraph subGraph1["Handler Selection Logic"]
    ITERATOR["Handler slice iterator"]
    FIRST["Take first handler"]
    WARN["Warn if multiple handlers"]
    DEFAULT["Default behavior if none"]
end
subgraph subGraph0["Dispatch Flow"]
    TRAP_EVENT["Trap EventHardware generated"]
    ARCH_HANDLER["Architecture HandlerCaptures context"]
    HANDLE_MACRO["handle_trap! macroFinds registered handler"]
    USER_HANDLER["User HandlerApplication logic"]
    RETURN["Return ControlResume or terminate"]
end

ARCH_HANDLER --> HANDLE_MACRO
HANDLE_MACRO --> ITERATOR
HANDLE_MACRO --> USER_HANDLER
ITERATOR --> DEFAULT
ITERATOR --> FIRST
ITERATOR --> WARN
TRAP_EVENT --> ARCH_HANDLER
USER_HANDLER --> RETURN
```

**Sources:** [src/trap.rs(L25 - L38)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/trap.rs#L25-L38) [src/trap.rs(L42 - L44)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/trap.rs#L42-L44)

## Architecture-Agnostic Abstractions

The library provides common abstractions that are implemented differently by each architecture but present a unified interface to higher-level code.

### Core Data Structure Patterns

Each architecture implements the same core abstractions with architecture-specific layouts:

```mermaid
flowchart TD
subgraph subGraph1["Architecture Implementations"]
    X86_IMPL["x86_64 ImplementationCallee-saved regs + stack + TLS"]
    ARM_IMPL["aarch64 ImplementationR19-R29 + SP + TPIDR + FpState"]
    RISC_IMPL["riscv Implementationra, sp, s0-s11, tp + SATP"]
    LOONG_IMPL["loongarch64 ImplementationCallee-saved + SP + TP + FPU"]
end
subgraph subGraph0["Common Abstractions"]
    TASK_CONTEXT["TaskContextMinimal context for task switching"]
    TRAP_FRAME["TrapFrameComplete CPU state for exceptions"]
    EXTENDED_STATE["Extended StateFPU, SIMD, special registers"]
end

EXTENDED_STATE --> ARM_IMPL
EXTENDED_STATE --> LOONG_IMPL
EXTENDED_STATE --> RISC_IMPL
EXTENDED_STATE --> X86_IMPL
TASK_CONTEXT --> ARM_IMPL
TASK_CONTEXT --> LOONG_IMPL
TASK_CONTEXT --> RISC_IMPL
TASK_CONTEXT --> X86_IMPL
TRAP_FRAME --> ARM_IMPL
TRAP_FRAME --> LOONG_IMPL
TRAP_FRAME --> RISC_IMPL
TRAP_FRAME --> X86_IMPL
```

**Sources:** [src/lib.rs(L14 - L28)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/lib.rs#L14-L28) [src/trap.rs(L5)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/trap.rs#L5-L5)

### Common Interface Design

The library uses Rust's module system and conditional compilation to provide a unified interface while allowing architecture-specific implementations:

```mermaid
flowchart TD
subgraph subGraph2["Architecture Modules"]
    X86_MOD["x86_64 moduleIntel/AMD implementation"]
    ARM_MOD["aarch64 moduleARM implementation"]
    RISC_MOD["riscv moduleRISC-V implementation"]
    LOONG_MOD["loongarch64 moduleLoongArch implementation"]
end
subgraph subGraph1["Architecture Selection"]
    CFG_IF["cfg_if! macroCompile-time arch selection"]
    TARGET_ARCH["target_arch attributesRust compiler directives"]
end
subgraph subGraph0["Interface Layer"]
    LIB_RS["src/lib.rsMain entry point"]
    TRAP_RS["src/trap.rsCross-arch trap handling"]
    PUBLIC_API["Public APIRe-exported types and functions"]
end

ARM_MOD --> PUBLIC_API
CFG_IF --> TARGET_ARCH
LIB_RS --> CFG_IF
LOONG_MOD --> PUBLIC_API
RISC_MOD --> PUBLIC_API
TARGET_ARCH --> ARM_MOD
TARGET_ARCH --> LOONG_MOD
TARGET_ARCH --> RISC_MOD
TARGET_ARCH --> X86_MOD
TRAP_RS --> PUBLIC_API
X86_MOD --> PUBLIC_API
```

**Sources:** [src/lib.rs(L14 - L28)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/lib.rs#L14-L28)

## Feature-Based Conditional Compilation

The axcpu library uses Cargo features to enable optional functionality across all architectures. These features control the compilation of additional capabilities that may not be needed in all use cases.

### Feature Flag System

|Feature|Purpose|Availability|
| --- | --- | --- |
|uspace|User space support including system calls|All architectures|
|fp-simd|Floating point and SIMD register management|All architectures|
|tls|Thread-local storage support|All architectures|

### User Space Feature Integration

The `uspace` feature enables system call handling across all architectures:

```mermaid
flowchart TD
subgraph subGraph1["Architecture Integration"]
    X86_SYSCALL["x86_64: SYSCALL instructionGS_BASE, CR3 management"]
    ARM_SYSCALL["aarch64: SVC instructionTTBR0 user page tables"]
    RISC_SYSCALL["riscv: ECALL instructionSATP register switching"]
    LOONG_SYSCALL["loongarch64: SYSCALLPGDL page directory"]
end
subgraph subGraph0["uspace Feature"]
    FEATURE_FLAG["#[cfg(feature = uspace)]Conditional compilation"]
    SYSCALL_SLICE["SYSCALL handler sliceSystem call dispatch"]
    HANDLE_SYSCALL_FN["handle_syscall functionDirect invocation"]
end

FEATURE_FLAG --> SYSCALL_SLICE
HANDLE_SYSCALL_FN --> ARM_SYSCALL
HANDLE_SYSCALL_FN --> LOONG_SYSCALL
HANDLE_SYSCALL_FN --> RISC_SYSCALL
HANDLE_SYSCALL_FN --> X86_SYSCALL
SYSCALL_SLICE --> HANDLE_SYSCALL_FN
```

**Sources:** [src/trap.rs(L19 - L22)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/trap.rs#L19-L22) [src/trap.rs(L41 - L44)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/trap.rs#L41-L44)

## Common Memory Management Abstractions

The library integrates with external crates to provide consistent memory management abstractions across architectures.

### External Dependencies Integration

```mermaid
flowchart TD
subgraph subGraph2["Architecture Implementation"]
    MMU_INIT["init_mmu functionsArchitecture-specific setup"]
    PAGE_TABLES["Page table managementArchitecture-specific formats"]
    TLB_MGMT["TLB managementTranslation cache control"]
end
subgraph subGraph1["Cross-Architecture Usage"]
    VIRT_ADDR["VirtAddrVirtual address type"]
    PAGE_FAULT_FLAGS["PageFaultFlagsAlias for MappingFlags"]
    TRAP_HANDLERS["Distributed trap handlersCompile-time registration"]
end
subgraph subGraph0["External Crate Integration"]
    MEMORY_ADDR["memory_addr crateVirtAddr, PhysAddr types"]
    PAGE_TABLE_ENTRY["page_table_entry crateMappingFlags abstraction"]
    LINKME["linkme crateDistributed slice collection"]
end

LINKME --> TRAP_HANDLERS
MEMORY_ADDR --> VIRT_ADDR
PAGE_FAULT_FLAGS --> PAGE_TABLES
PAGE_TABLE_ENTRY --> PAGE_FAULT_FLAGS
TRAP_HANDLERS --> TLB_MGMT
VIRT_ADDR --> MMU_INIT
```

**Sources:** [src/lib.rs(L9)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/lib.rs#L9-L9) [src/trap.rs(L3)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/trap.rs#L3-L3) [src/trap.rs(L8)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/trap.rs#L8-L8) [src/trap.rs(L6)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/trap.rs#L6-L6)

## Error Handling and Warnings

The cross-architecture framework includes built-in error handling and diagnostic capabilities that work consistently across all supported architectures.

### Handler Registration Validation

The trap handling system validates handler registration at runtime and provides warnings for common configuration issues:

```mermaid
flowchart TD
subgraph subGraph0["Handler Validation Flow"]
    CHECK_ITER["Check handler iterator"]
    FIRST_HANDLER["Get first handler"]
    CHECK_MORE["Check for additional handlers"]
    WARN_MULTIPLE["Warn about multiple handlers"]
    INVOKE["Invoke selected handler"]
    NO_HANDLER["Warn about missing handlers"]
end

CHECK_ITER --> FIRST_HANDLER
CHECK_ITER --> NO_HANDLER
CHECK_MORE --> INVOKE
CHECK_MORE --> WARN_MULTIPLE
FIRST_HANDLER --> CHECK_MORE
NO_HANDLER --> INVOKE
```

The system issues specific warning messages for:

* Multiple handlers registered for the same trap type
* No handlers registered for a trap that occurs
* Feature mismatches between compile-time and runtime expectations

**Sources:** [src/trap.rs(L25 - L38)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/trap.rs#L25-L38)