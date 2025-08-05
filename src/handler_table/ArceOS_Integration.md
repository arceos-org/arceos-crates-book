# ArceOS Integration

> **Relevant source files**
> * [Cargo.toml](https://github.com/arceos-org/handler_table/blob/036a12c4/Cargo.toml)

This document explains how the `handler_table` crate integrates into the ArceOS operating system ecosystem and serves as a foundational component for lock-free event handling in kernel and embedded environments.

The scope covers the architectural role of `handler_table` within ArceOS, its `no_std` design constraints, and how it fits into the broader operating system infrastructure. For details about the lock-free programming concepts and performance benefits, see [Lock-free Design Benefits](/arceos-org/handler_table/1.2-lock-free-design-benefits). For implementation specifics, see [Implementation Details](/arceos-org/handler_table/3-implementation-details).

## Purpose and Role in ArceOS

The `handler_table` crate provides a core event handling mechanism designed specifically for the ArceOS operating system. ArceOS is a modular, component-based operating system written in Rust that targets both traditional and embedded computing environments.

**ArceOS Integration Architecture**

```mermaid
flowchart TD
subgraph Event_Sources["Event Sources"]
    Hardware_Interrupts["Hardware Interrupts"]
    Timer_Events["Timer Events"]
    System_Calls["System Calls"]
    IO_Completion["I/O Completion"]
end
subgraph handler_table_Crate["handler_table Crate"]
    HandlerTable_Struct["HandlerTable"]
    register_handler["register_handler()"]
    handle_method["handle()"]
    unregister_handler["unregister_handler()"]
end
subgraph ArceOS_Ecosystem["ArceOS Operating System Ecosystem"]
    ArceOS_Kernel["ArceOS Kernel Core"]
    Task_Scheduler["Task Scheduler"]
    Memory_Manager["Memory Manager"]
    Device_Drivers["Device Drivers"]
    Interrupt_Controller["Interrupt Controller"]
end

ArceOS_Kernel --> HandlerTable_Struct
Device_Drivers --> HandlerTable_Struct
HandlerTable_Struct --> handle_method
Hardware_Interrupts --> register_handler
IO_Completion --> register_handler
Interrupt_Controller --> HandlerTable_Struct
System_Calls --> register_handler
Task_Scheduler --> HandlerTable_Struct
Timer_Events --> register_handler
handle_method --> unregister_handler
```

The crate serves as a central dispatch mechanism that allows different ArceOS subsystems to register event handlers without requiring traditional locking mechanisms that would introduce latency and complexity in kernel code.

Sources: [Cargo.toml(L1 - L15)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/Cargo.toml#L1-L15)

## No-std Environment Design

The `handler_table` crate is specifically designed for `no_std` environments, making it suitable for embedded systems and kernel-level code where the standard library is not available.

|Design Constraint|Implementation|ArceOS Benefit|
| --- | --- | --- |
|No heap allocation|Fixed-size arrays with compile-time bounds|Predictable memory usage in kernel|
|No standard library|Core atomics and primitive types only|Minimal dependencies for embedded targets|
|Zero-cost abstractions|Direct atomic operations without runtime overhead|High-performance event handling|
|Compile-time sizing|GenericHandlerTable<N>with const parameter|Memory layout known at build time|

**No-std Integration Flow**

```mermaid
flowchart TD
subgraph ArceOS_Targets["ArceOS Target Architectures"]
    x86_64_none["x86_64-unknown-none"]
    riscv64_none["riscv64gc-unknown-none-elf"]
    aarch64_none["aarch64-unknown-none-softfloat"]
end
subgraph handler_table_Integration["handler_table Integration"]
    Core_Only["core:: imports only"]
    AtomicUsize_Array["AtomicUsize array storage"]
    Const_Generic["HandlerTable sizing"]
end
subgraph Build_Process["ArceOS Build Process"]
    Cargo_Config["Cargo Configuration"]
    Target_Spec["Target Specification"]
    no_std_Flag["#![no_std] Attribute"]
end

AtomicUsize_Array --> Const_Generic
Cargo_Config --> Core_Only
Const_Generic --> aarch64_none
Const_Generic --> riscv64_none
Const_Generic --> x86_64_none
Core_Only --> AtomicUsize_Array
Target_Spec --> Core_Only
no_std_Flag --> Core_Only
```

This design enables ArceOS to use `handler_table` across different target architectures without modification, supporting both traditional x86_64 systems and embedded RISC-V and ARM platforms.

Sources: [Cargo.toml(L12)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/Cargo.toml#L12-L12)

## Integration Patterns

ArceOS components integrate with `handler_table` through several common patterns that leverage the crate's lock-free design.

**Event Registration Pattern**

```mermaid
sequenceDiagram
    participant ArceOSModule as "ArceOS Module"
    participant HandlerTableN as "HandlerTable<N>"
    participant AtomicUsizeidx as "AtomicUsize[idx]"

    Note over ArceOSModule,AtomicUsizeidx: Module Initialization Phase
    ArceOSModule ->> HandlerTableN: "register_handler(event_id, handler_fn)"
    HandlerTableN ->> AtomicUsizeidx: "compare_exchange(0, handler_ptr)"
    alt Registration Success
        AtomicUsizeidx -->> HandlerTableN: "Ok(0)"
        HandlerTableN -->> ArceOSModule: "true"
        Note over ArceOSModule: "Handler registered successfully"
    else Slot Already Occupied
        AtomicUsizeidx -->> HandlerTableN: "Err(existing_ptr)"
        HandlerTableN -->> ArceOSModule: "false"
        Note over ArceOSModule: "Handle registration conflict"
    end
    Note over ArceOSModule,AtomicUsizeidx: Runtime Event Handling
    ArceOSModule ->> HandlerTableN: "handle(event_id)"
    HandlerTableN ->> AtomicUsizeidx: "load(Ordering::Acquire)"
    AtomicUsizeidx -->> HandlerTableN: "handler_ptr"
    alt Handler Exists
        HandlerTableN ->> HandlerTableN: "transmute & call handler"
        HandlerTableN -->> ArceOSModule: "true (handled)"
    else No Handler
        HandlerTableN -->> ArceOSModule: "false (not handled)"
    end
```

This pattern allows ArceOS modules to register event handlers during system initialization and efficiently dispatch events during runtime without locks.

Sources: [Cargo.toml(L6)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/Cargo.toml#L6-L6)

## Repository and Documentation Structure

The crate is maintained as part of the broader ArceOS organization, with integration points designed to support the operating system's modular architecture.

**ArceOS Organization Structure**

```mermaid
flowchart TD
subgraph Integration_Points["Integration Points"]
    homepage_link["Homepage -> arceos repo"]
    keywords_arceos["Keywords: 'arceos'"]
    license_compat["Compatible licensing"]
end
subgraph Documentation["Documentation Infrastructure"]
    docs_rs["docs.rs documentation"]
    github_pages["GitHub Pages"]
    readme_docs["README.md examples"]
end
subgraph arceos_org["ArceOS Organization (GitHub)"]
    arceos_main["arceos (main repository)"]
    handler_table_repo["handler_table (this crate)"]
    other_crates["Other ArceOS crates"]
end

arceos_main --> handler_table_repo
handler_table_repo --> docs_rs
handler_table_repo --> github_pages
handler_table_repo --> other_crates
handler_table_repo --> readme_docs
homepage_link --> arceos_main
keywords_arceos --> arceos_main
license_compat --> arceos_main
```

The repository structure ensures that `handler_table` can be discovered and integrated by ArceOS developers while maintaining independent versioning and documentation.

Sources: [Cargo.toml(L8 - L11)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/Cargo.toml#L8-L11)

## Version and Compatibility

The current version `0.1.2` indicates this is an early-stage crate that is being actively developed alongside the ArceOS ecosystem. The version strategy supports incremental improvements while maintaining API stability for core ArceOS components.

|Aspect|Current State|ArceOS Integration Impact|
| --- | --- | --- |
|Version|0.1.2|Early development, compatible with ArceOS components|
|License|GPL-3.0-or-later OR Apache-2.0 OR MulanPSL-2.0|Multiple license options for different ArceOS use cases|
|Edition|2021|Modern Rust features available for ArceOS development|
|Dependencies|None|Zero-dependency design reduces ArceOS build complexity|

The tri-license approach provides flexibility for different ArceOS deployment scenarios, supporting both open-source and commercial use cases.

Sources: [Cargo.toml(L3 - L7)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/Cargo.toml#L3-L7) [Cargo.toml(L14 - L15)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/Cargo.toml#L14-L15)