# AArch64 Trap and Exception Handling

> **Relevant source files**
> * [src/aarch64/trap.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/trap.rs)

This document covers the AArch64 (ARM64) trap and exception handling implementation in the axcpu library. It details the exception classification system, handler dispatch mechanism, and integration with the cross-architecture trap handling framework. For AArch64 context management including `TrapFrame` structure, see [AArch64 Context Management](/arceos-org/axcpu/3.1-aarch64-context-management). For system initialization and exception vector setup, see [AArch64 System Initialization](/arceos-org/axcpu/3.3-aarch64-system-initialization).

## Exception Classification System

The AArch64 trap handler uses a two-dimensional classification system to categorize exceptions based on their type and source context.

### Exception Types

The system defines four primary exception types in the `TrapKind` enumeration:

|Exception Type|Value|Description|
| --- | --- | --- |
|Synchronous|0|Synchronous exceptions requiring immediate handling|
|Irq|1|Standard interrupt requests|
|Fiq|2|Fast interrupt requests (higher priority)|
|SError|3|System error exceptions|

### Exception Sources

The `TrapSource` enumeration categorizes the execution context from which exceptions originate:

|Source Type|Value|Description|
| --- | --- | --- |
|CurrentSpEl0|0|Current stack pointer, Exception Level 0|
|CurrentSpElx|1|Current stack pointer, Exception Level 1+|
|LowerAArch64|2|Lower exception level, AArch64 state|
|LowerAArch32|3|Lower exception level, AArch32 state|

Sources: [src/aarch64/trap.rs(L9 - L27)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/trap.rs#L9-L27)

## Exception Handler Architecture

### Exception Handler Dispatch Flow

```mermaid
flowchart TD
ASM_ENTRY["trap.S Assembly Entry Points"]
DISPATCH["Exception Dispatch Logic"]
SYNC_CHECK["Exception Type?"]
SYNC_HANDLER["handle_sync_exception()"]
IRQ_HANDLER["handle_irq_exception()"]
INVALID_HANDLER["invalid_exception()"]
ESR_READ["Read ESR_EL1 Register"]
EC_DECODE["Decode Exception Class"]
SYSCALL["handle_syscall()"]
INSTR_ABORT["handle_instruction_abort()"]
DATA_ABORT["handle_data_abort()"]
BRK_HANDLER["Debug Break Handler"]
PANIC["Panic with Details"]
TRAP_MACRO["handle_trap!(IRQ, 0)"]
PAGE_FAULT_CHECK["Translation/Permission Fault?"]
PAGE_FAULT_CHECK2["Translation/Permission Fault?"]
PAGE_FAULT_HANDLER["handle_trap!(PAGE_FAULT, ...)"]
ABORT_PANIC["Panic with Fault Details"]

ASM_ENTRY --> DISPATCH
DATA_ABORT --> PAGE_FAULT_CHECK2
DISPATCH --> SYNC_CHECK
EC_DECODE --> BRK_HANDLER
EC_DECODE --> DATA_ABORT
EC_DECODE --> INSTR_ABORT
EC_DECODE --> PANIC
EC_DECODE --> SYSCALL
ESR_READ --> EC_DECODE
INSTR_ABORT --> PAGE_FAULT_CHECK
IRQ_HANDLER --> TRAP_MACRO
PAGE_FAULT_CHECK --> ABORT_PANIC
PAGE_FAULT_CHECK --> PAGE_FAULT_HANDLER
PAGE_FAULT_CHECK2 --> ABORT_PANIC
PAGE_FAULT_CHECK2 --> PAGE_FAULT_HANDLER
SYNC_CHECK --> INVALID_HANDLER
SYNC_CHECK --> IRQ_HANDLER
SYNC_CHECK --> SYNC_HANDLER
SYNC_HANDLER --> ESR_READ
```

Sources: [src/aarch64/trap.rs(L29 - L121)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/trap.rs#L29-L121)

### Core Handler Functions

The trap handling system implements several specialized handler functions:

#### Synchronous Exception Handler

The `handle_sync_exception` function serves as the primary dispatcher for synchronous exceptions, using the ESR_EL1 register to determine the specific exception class:

```mermaid
flowchart TD
ESR_READ["ESR_EL1.extract()"]
EC_MATCH["Match Exception Class"]
SYSCALL_IMPL["tf.r[0] = handle_syscall(tf, tf.r[8])"]
INSTR_LOWER["handle_instruction_abort(tf, iss, true)"]
INSTR_CURRENT["handle_instruction_abort(tf, iss, false)"]
DATA_LOWER["handle_data_abort(tf, iss, true)"]
DATA_CURRENT["handle_data_abort(tf, iss, false)"]
BRK_ADVANCE["tf.elr += 4"]
UNHANDLED_PANIC["panic!(unhandled exception)"]

EC_MATCH --> BRK_ADVANCE
EC_MATCH --> DATA_CURRENT
EC_MATCH --> DATA_LOWER
EC_MATCH --> INSTR_CURRENT
EC_MATCH --> INSTR_LOWER
EC_MATCH --> SYSCALL_IMPL
EC_MATCH --> UNHANDLED_PANIC
ESR_READ --> EC_MATCH
```

Sources: [src/aarch64/trap.rs(L94 - L121)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/trap.rs#L94-L121)

#### IRQ Exception Handler

The `handle_irq_exception` function provides a simple interface to the cross-architecture interrupt handling system through the `handle_trap!` macro:

Sources: [src/aarch64/trap.rs(L37 - L40)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/trap.rs#L37-L40)

#### Invalid Exception Handler

The `invalid_exception` function handles unexpected exception types or sources by panicking with detailed context information including the `TrapFrame`, exception kind, and source classification:

Sources: [src/aarch64/trap.rs(L29 - L35)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/trap.rs#L29-L35)

## Memory Fault Handling

The AArch64 implementation provides specialized handling for instruction and data aborts, which correspond to page faults in other architectures.

### Page Fault Classification

Both instruction and data aborts use a common pattern for classifying page faults:

|Fault Type|ISS Bits|Handled|Description|
| --- | --- | --- | --- |
|Translation Fault|0b0100|Yes|Missing page table entry|
|Permission Fault|0b1100|Yes|Access permission violation|
|Other Faults|Various|No|Unhandled fault types|

### Access Flag Generation

The system generates `PageFaultFlags` based on the fault characteristics:

#### Instruction Abort Flags

* Always includes `PageFaultFlags::EXECUTE`
* Adds `PageFaultFlags::USER` for EL0 (user) faults

#### Data Abort Flags

* Uses ESR_EL1 ISS WnR bit (Write not Read) to determine access type
* Considers CM bit (Cache Maintenance) to exclude cache operations from write classification
* Adds `PageFaultFlags::USER` for EL0 (user) faults

```mermaid
flowchart TD
DATA_ABORT["Data Abort Exception"]
ISS_DECODE["Decode ISS Bits from ESR_EL1"]
WNR_CHECK["WnR Bit Set?"]
CM_CHECK["CM Bit Set?"]
EL_CHECK["Exception from EL0?"]
WRITE_FLAG["PageFaultFlags::WRITE"]
READ_FLAG["PageFaultFlags::read"]
OVERRIDE_READ["Override to READ (cache maintenance)"]
USER_FLAG["Add PageFaultFlags::USER"]
ACCESS_FLAGS["Combined Access Flags"]
read_FLAG["PageFaultFlags::read"]
DFSC_CHECK["DFSC bits = 0100 or 1100?"]
PAGE_FAULT_TRAP["handle_trap!(PAGE_FAULT, vaddr, flags, is_user)"]
UNHANDLED_PANIC["panic!(unhandled data abort)"]

ACCESS_FLAGS --> DFSC_CHECK
CM_CHECK --> OVERRIDE_READ
DATA_ABORT --> ISS_DECODE
DFSC_CHECK --> PAGE_FAULT_TRAP
DFSC_CHECK --> UNHANDLED_PANIC
EL_CHECK --> USER_FLAG
ISS_DECODE --> CM_CHECK
ISS_DECODE --> EL_CHECK
ISS_DECODE --> WNR_CHECK
OVERRIDE_READ --> ACCESS_FLAGS
USER_FLAG --> ACCESS_FLAGS
WNR_CHECK --> READ_FLAG
WNR_CHECK --> WRITE_FLAG
WRITE_FLAG --> ACCESS_FLAGS
read_FLAG --> ACCESS_FLAGS
```

Sources: [src/aarch64/trap.rs(L42 - L92)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/trap.rs#L42-L92)

## System Call Integration

When the `uspace` feature is enabled, the trap handler supports AArch64 system calls through the SVC (Supervisor Call) instruction.

### System Call Convention

* System call number: Passed in register `x8` (`tf.r[8]`)
* Return value: Stored in register `x0` (`tf.r[0]`)
* Handler: Delegates to cross-architecture `crate::trap::handle_syscall`

Sources: [src/aarch64/trap.rs(L99 - L102)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/trap.rs#L99-L102)

## Integration with Cross-Architecture Framework

The AArch64 trap handler integrates with the broader axcpu trap handling framework through several mechanisms:

### Trap Handler Macros

The implementation uses the `handle_trap!` macro for consistent integration:

* `handle_trap!(IRQ, 0)` for interrupt processing
* `handle_trap!(PAGE_FAULT, vaddr, access_flags, is_user)` for memory faults

### Assembly Integration

The Rust handlers are called from assembly code included via:

```yaml
core::arch::global_asm!(include_str!("trap.S"));
```

### Register Access

The system uses the `aarch64_cpu` crate to access AArch64 system registers:

* `ESR_EL1`: Exception Syndrome Register for fault classification
* `FAR_EL1`: Fault Address Register for memory fault addresses

Sources: [src/aarch64/trap.rs(L1 - L7)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/trap.rs#L1-L7)