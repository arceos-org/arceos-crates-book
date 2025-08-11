# x86_64 System Initialization

> **Relevant source files**
> * [src/x86_64/gdt.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/gdt.rs)
> * [src/x86_64/idt.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/idt.rs)
> * [src/x86_64/init.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/init.rs)

This document covers the x86_64 CPU initialization procedures during system bootstrap, including the setup of critical system tables and per-CPU data structures. The initialization process establishes the Global Descriptor Table (GDT), Interrupt Descriptor Table (IDT), and prepares the CPU for trap handling and task switching.

For x86_64 trap and exception handling mechanisms, see [x86_64 Trap and Exception Handling](/arceos-org/axcpu/2.2-x86_64-trap-and-exception-handling). For x86_64 system call implementation details, see [x86_64 System Calls](/arceos-org/axcpu/2.3-x86_64-system-calls).

## Initialization Overview

The x86_64 system initialization follows a structured sequence that prepares the CPU for operation in both kernel and user modes. The process is coordinated through the `init` module and involves setting up fundamental x86_64 system tables.

```mermaid
flowchart TD
start["System Boot"]
init_percpu["init_percpu()"]
init_trap["init_trap()"]
init_gdt["init_gdt()"]
init_idt["init_idt()"]
init_syscall["init_syscall()"]
gdt_setup["GDT Table Setup"]
load_gdt["lgdt instruction"]
load_tss["load_tss()"]
idt_setup["IDT Table Setup"]
load_idt["lidt instruction"]
msr_setup["MSR Configuration"]
ready["CPU Ready"]

gdt_setup --> load_gdt
idt_setup --> load_idt
init_gdt --> gdt_setup
init_idt --> idt_setup
init_percpu --> init_trap
init_syscall --> msr_setup
init_trap --> init_gdt
init_trap --> init_idt
init_trap --> init_syscall
load_gdt --> load_tss
load_idt --> ready
load_tss --> ready
msr_setup --> ready
start --> init_percpu
```

**Initialization Sequence Flow**

Sources: [src/x86_64/init.rs(L15 - L38)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/init.rs#L15-L38)

## Per-CPU Data Structure Initialization

The `init_percpu()` function establishes per-CPU data structures required for multi-core operation. This initialization must occur before trap handling setup.

|Function|Purpose|Dependencies|
| --- | --- | --- |
|percpu::init()|Initialize percpu framework|None|
|percpu::init_percpu_reg()|Set CPU-specific register|CPU ID|

The function takes a `cpu_id` parameter to configure CPU-specific storage and must be called on each CPU core before `init_trap()`.

```mermaid
flowchart TD
percpu_init["percpu::init()"]
percpu_reg["percpu::init_percpu_reg(cpu_id)"]
tss_ready["TSS per-CPU ready"]
gdt_ready["GDT per-CPU ready"]

percpu_init --> percpu_reg
percpu_reg --> gdt_ready
percpu_reg --> tss_ready
```

**Per-CPU Initialization Dependencies**

Sources: [src/x86_64/init.rs(L15 - L18)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/init.rs#L15-L18)

## Global Descriptor Table (GDT) Initialization

The GDT setup establishes memory segmentation for x86_64 operation, including kernel and user code/data segments plus the Task State Segment (TSS).

### GDT Structure

The `GdtStruct` contains predefined segment selectors for different privilege levels and operating modes:

|Selector|Index|Purpose|Privilege|
| --- | --- | --- | --- |
|KCODE32_SELECTOR|1|Kernel 32-bit code|Ring 0|
|KCODE64_SELECTOR|2|Kernel 64-bit code|Ring 0|
|KDATA_SELECTOR|3|Kernel data|Ring 0|
|UCODE32_SELECTOR|4|User 32-bit code|Ring 3|
|UDATA_SELECTOR|5|User data|Ring 3|
|UCODE64_SELECTOR|6|User 64-bit code|Ring 3|
|TSS_SELECTOR|7|Task State Segment|Ring 0|

### GDT Initialization Process

```mermaid
flowchart TD
tss_percpu["TSS per-CPU instance"]
gdt_new["GdtStruct::new(tss)"]
gdt_table["table[1-8] segment descriptors"]
gdt_load["gdt.load()"]
lgdt["lgdt instruction"]
cs_set["CS::set_reg(KCODE64_SELECTOR)"]
load_tss_call["gdt.load_tss()"]
ltr["load_tss(TSS_SELECTOR)"]

cs_set --> load_tss_call
gdt_load --> lgdt
gdt_new --> gdt_table
gdt_table --> gdt_load
lgdt --> cs_set
load_tss_call --> ltr
tss_percpu --> gdt_new
```

**GDT Setup and Loading Process**

The TSS is configured as a per-CPU structure using the `percpu` crate, allowing each CPU core to maintain its own stack pointers and state information.

Sources: [src/x86_64/gdt.rs(L104 - L111)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/gdt.rs#L104-L111) [src/x86_64/gdt.rs(L41 - L55)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/gdt.rs#L41-L55) [src/x86_64/gdt.rs(L73 - L90)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/gdt.rs#L73-L90)

## Interrupt Descriptor Table (IDT) Initialization

The IDT setup configures interrupt and exception handlers for the x86_64 architecture, mapping 256 possible interrupt vectors to their respective handler functions.

### IDT Structure and Handler Mapping

The `IdtStruct` populates all 256 interrupt vectors from an external `trap_handler_table`:

```mermaid
flowchart TD
trap_handler_table["trap_handler_table[256]"]
idt_new["IdtStruct::new()"]
entry_loop["for i in 0..256"]
set_handler["entries[i].set_handler_fn()"]
privilege_check["i == 0x3 || i == 0x80?"]
ring3["set_privilege_level(Ring3)"]
ring0["Default Ring0"]
idt_load["idt.load()"]
lidt["lidt instruction"]

entry_loop --> set_handler
idt_load --> lidt
idt_new --> entry_loop
privilege_check --> ring0
privilege_check --> ring3
ring0 --> idt_load
ring3 --> idt_load
set_handler --> privilege_check
trap_handler_table --> idt_new
```

**IDT Initialization and Handler Assignment**

### Special Privilege Level Handling

Two interrupt vectors receive special Ring 3 (user mode) access privileges:

* Vector `0x3`: Breakpoint exception for user-space debugging
* Vector `0x80`: Legacy system call interface

Sources: [src/x86_64/idt.rs(L22 - L46)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/idt.rs#L22-L46) [src/x86_64/idt.rs(L77 - L81)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/idt.rs#L77-L81)

## System Call Initialization (Optional)

When the `uspace` feature is enabled, `init_syscall()` configures Model-Specific Registers (MSRs) for fast system call handling using the `syscall`/`sysret` instructions.

```

```

**Conditional System Call Initialization**

The system call initialization is conditionally compiled and only executed when user space support is required.

Sources: [src/x86_64/init.rs(L6 - L7)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/init.rs#L6-L7) [src/x86_64/init.rs(L36 - L37)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/init.rs#L36-L37)

## Task State Segment (TSS) Management

The TSS provides crucial stack switching capabilities for privilege level transitions, particularly important for user space support.

### TSS Per-CPU Organization

```mermaid
flowchart TD
percpu_tss["#[percpu::def_percpu] TSS"]
cpu0_tss["CPU 0 TSS instance"]
cpu1_tss["CPU 1 TSS instance"]
cpun_tss["CPU N TSS instance"]
rsp0_0["privilege_stack_table[0]"]
rsp0_1["privilege_stack_table[0]"]
rsp0_n["privilege_stack_table[0]"]

cpu0_tss --> rsp0_0
cpu1_tss --> rsp0_1
cpun_tss --> rsp0_n
percpu_tss --> cpu0_tss
percpu_tss --> cpu1_tss
percpu_tss --> cpun_tss
```

**Per-CPU TSS Instance Management**

The TSS includes utility functions for managing the Ring 0 stack pointer (`RSP0`) used during privilege level transitions:

* `read_tss_rsp0()`: Retrieves current RSP0 value
* `write_tss_rsp0()`: Updates RSP0 for kernel stack switching

Sources: [src/x86_64/gdt.rs(L10 - L12)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/gdt.rs#L10-L12) [src/x86_64/gdt.rs(L114 - L129)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/gdt.rs#L114-L129)

## Complete Initialization Function

The `init_trap()` function orchestrates the complete x86_64 system initialization sequence:

```rust
pub fn init_trap() {
    init_gdt();
    init_idt();
    #[cfg(feature = "uspace")]
    init_syscall();
}
```

This function must be called after `init_percpu()` to ensure proper per-CPU data structure availability. The initialization establishes a fully functional x86_64 CPU state capable of handling interrupts, exceptions, and system calls.

Sources: [src/x86_64/init.rs(L33 - L38)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/x86_64/init.rs#L33-L38)