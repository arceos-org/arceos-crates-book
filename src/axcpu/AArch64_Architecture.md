# AArch64 Architecture

> **Relevant source files**
> * [src/aarch64/mod.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/mod.rs)

## Purpose and Scope

This document provides a comprehensive overview of AArch64 (ARM64) architecture support within the axcpu library. The AArch64 implementation provides CPU abstraction capabilities including context management, trap handling, and system initialization for ARM 64-bit processors.

For detailed information about specific AArch64 subsystems, see [AArch64 Context Management](/arceos-org/axcpu/3.1-aarch64-context-management), [AArch64 Trap and Exception Handling](/arceos-org/axcpu/3.2-aarch64-trap-and-exception-handling), and [AArch64 System Initialization](/arceos-org/axcpu/3.3-aarch64-system-initialization). For information about cross-architecture features that work with AArch64, see [Cross-Architecture Features](/arceos-org/axcpu/6-cross-architecture-features).

## Module Organization

The AArch64 architecture support is organized into several focused modules that handle different aspects of CPU management and abstraction.

### AArch64 Module Structure

```

```

The module structure follows a consistent pattern with other architectures in axcpu, providing specialized implementations for AArch64's unique characteristics while maintaining a unified interface.

Sources: [src/aarch64/mod.rs(L1 - L13)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/mod.rs#L1-L13)

## Core Data Structures

The AArch64 implementation centers around three primary data structures that encapsulate different aspects of CPU state management.

### Context Management Abstractions

```mermaid
flowchart TD
subgraph subGraph3["FpState Contents"]
    FP_VREGS["V0-V31Vector registers"]
    FP_FPCR["FPCRFloating-point control"]
    FP_FPSR["FPSRFloating-point status"]
end
subgraph subGraph2["TrapFrame Contents"]
    TF_REGS["R0-R30General purpose registers"]
    TF_SP_USER["SP_EL0User stack pointer"]
    TF_ELR["ELR_EL1Exception link register"]
    TF_SPSR["SPSR_EL1Saved program status"]
end
subgraph subGraph1["TaskContext Contents"]
    TC_REGS["R19-R29Callee-saved registers"]
    TC_SP["SPStack pointer"]
    TC_TPIDR["TPIDR_EL0Thread pointer"]
    TC_FP_STATE["FpStateFloating-point context"]
end
subgraph subGraph0["AArch64 Context Types"]
    TC["TaskContext"]
    TF["TrapFrame"]
    FP["FpState"]
end

FP --> FP_FPCR
FP --> FP_FPSR
FP --> FP_VREGS
TC --> TC_FP_STATE
TC --> TC_REGS
TC --> TC_SP
TC --> TC_TPIDR
TC_FP_STATE --> FP
TF --> TF_ELR
TF --> TF_REGS
TF --> TF_SPSR
TF --> TF_SP_USER
```

These structures provide the foundation for AArch64's context switching, exception handling, and floating-point state management capabilities.

Sources: [src/aarch64/mod.rs(L12)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/mod.rs#L12-L12)

## Feature-Based Compilation

The AArch64 implementation uses conditional compilation to provide different capabilities based on the target environment and enabled features.

|Condition|Module|Purpose|
| --- | --- | --- |
|target_os = "none"|trap|Bare-metal trap handling|
|feature = "uspace"|uspace|User space support and system calls|
|Default|context,asm,init|Core functionality always available|

### Conditional Module Integration

```

```

This modular design allows the AArch64 implementation to be tailored for different deployment scenarios, from bare-metal embedded systems to full operating system kernels with user space support.

Sources: [src/aarch64/mod.rs(L6 - L10)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/mod.rs#L6-L10)

## Architecture Integration

The AArch64 implementation integrates with the broader axcpu framework through standardized interfaces and common abstractions.

### Cross-Architecture Compatibility

```mermaid
flowchart TD
subgraph subGraph2["External Dependencies"]
    AARCH64_CPU["aarch64-cpu crateArchitecture-specific types"]
    MEMORY_ADDR["memory_addrAddress abstractions"]
    PAGE_TABLE["page_table_entryPage table support"]
end
subgraph subGraph1["AArch64 Implementation"]
    AARCH64_CONTEXT["AArch64 ContextTaskContext, TrapFrame, FpState"]
    AARCH64_ASM["AArch64 AssemblyLow-level operations"]
    AARCH64_INIT["AArch64 InitSystem startup"]
    AARCH64_TRAP["AArch64 TrapException handlers"]
end
subgraph subGraph0["axcpu Framework"]
    CORE_TRAITS["Core TraitsCommon interfaces"]
    MEMORY_MGMT["Memory ManagementPage tables, MMU"]
    TRAP_FRAMEWORK["Trap FrameworkUnified exception handling"]
end

AARCH64_ASM --> AARCH64_CPU
AARCH64_CONTEXT --> AARCH64_CPU
AARCH64_INIT --> MEMORY_ADDR
AARCH64_INIT --> PAGE_TABLE
CORE_TRAITS --> AARCH64_CONTEXT
MEMORY_MGMT --> AARCH64_INIT
TRAP_FRAMEWORK --> AARCH64_TRAP
```

The AArch64 implementation follows the established patterns used by other architectures in axcpu, ensuring consistent behavior and interfaces across different processor architectures while leveraging AArch64-specific features and optimizations.

Sources: [src/aarch64/mod.rs(L1 - L13)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/mod.rs#L1-L13)