# ArceOS Crates Book

# cap_access

- [Overview](./cap_access/Overview.md)

- [Core Architecture](./cap_access/Core_Architecture.md)
    - [Capability System](./cap_access/Capability_System.md)
    - [Object Protection with WithCap](./cap_access/Object_Protection_with_WithCap.md)
    - [Access Control Methods](./cap_access/Access_Control_Methods.md)

- [Usage Guide](./cap_access/Usage_Guide.md)

- [ArceOS Integration](./cap_access/ArceOS_Integration.md)

- [Development Guide](./cap_access/Development_Guide.md)
    - [Build System and CI](./cap_access/Build_System_and_CI.md)
    - [Multi-Platform Support](./cap_access/Multi-Platform_Support.md)

# kspin

- [Overview](./kspin/Overview.md)
    - [Project Structure and Dependencies](./kspin/Project_Structure_and_Dependencies.md)

- [Spinlock Types and Public API](./kspin/Spinlock_Types_and_Public_API.md)
    - [SpinRaw](./kspin/SpinRaw.md)
    - [SpinNoPreempt](./kspin/SpinNoPreempt.md)
    - [SpinNoIrq](./kspin/SpinNoIrq.md)
    - [Usage Guidelines and Safety](./kspin/Usage_Guidelines_and_Safety.md)

- [Core Implementation Architecture](./kspin/Core_Implementation_Architecture.md)
    - [BaseSpinLock and BaseSpinLockGuard](./kspin/BaseSpinLock_and_BaseSpinLockGuard.md)
    - [BaseGuard Trait System](./kspin/BaseGuard_Trait_System.md)
    - [SMP vs Single-Core Implementation](./kspin/SMP_vs_Single-Core_Implementation.md)
    - [Memory Ordering and Atomic Operations](./kspin/Memory_Ordering_and_Atomic_Operations.md)

- [Development and Building](./kspin/Development_and_Building.md)
    - [Build System and Feature Flags](./kspin/Build_System_and_Feature_Flags.md)
    - [Testing and CI Pipeline](./kspin/Testing_and_CI_Pipeline.md)
    - [Development Environment Setup](./kspin/Development_Environment_Setup.md)

# kernel_guard

- [Overview](./kernel_guard/Overview.md)

- [Core Architecture](./kernel_guard/Core_Architecture.md)
    - [RAII Guards](./kernel_guard/RAII_Guards.md)
    - [Trait System](./kernel_guard/Trait_System.md)

- [Multi-Architecture Support](./kernel_guard/Multi-Architecture_Support.md)
    - [Architecture Abstraction Layer](./kernel_guard/Architecture_Abstraction_Layer.md)
    - [x86/x86_64 Implementation](./kernel_guard/x86_x86_64_Implementation.md)
    - [RISC-V Implementation](./kernel_guard/RISC-V_Implementation.md)
    - [AArch64 Implementation](./kernel_guard/AArch64_Implementation.md)
    - [LoongArch64 Implementation](./kernel_guard/LoongArch64_Implementation.md)

- [Integration Guide](./kernel_guard/Integration_Guide.md)
    - [Feature Configuration](./kernel_guard/Feature_Configuration.md)
    - [Implementing KernelGuardIf](./kernel_guard/Implementing_KernelGuardIf.md)

- [Development](./kernel_guard/Development.md)
    - [Build System and CI](./kernel_guard/Build_System_and_CI.md)
    - [Development Environment](./kernel_guard/Development_Environment.md)

# timer_list

- [Overview](./timer_list/Overview.md)

- [Core API Reference](./timer_list/Core_API_Reference.md)
    - [TimerList Data Structure](./timer_list/TimerList_Data_Structure.md)
    - [TimerEvent System](./timer_list/TimerEvent_System.md)

- [Usage Guide and Examples](./timer_list/Usage_Guide_and_Examples.md)

- [Development Workflow](./timer_list/Development_Workflow.md)
    - [Building and Testing](./timer_list/Building_and_Testing.md)
    - [Project Structure](./timer_list/Project_Structure.md)

# slab_allocator

- [Overview](./slab_allocator/Overview.md)

- [Getting Started](./slab_allocator/Getting_Started.md)

- [Core Architecture](./slab_allocator/Core_Architecture.md)
    - [Heap Allocator Design](./slab_allocator/Heap_Allocator_Design.md)
    - [Slab Implementation](./slab_allocator/Slab_Implementation.md)

- [API Reference](./slab_allocator/API_Reference.md)

- [Testing and Validation](./slab_allocator/Testing_and_Validation.md)

- [Development Workflow](./slab_allocator/Development_Workflow.md)

# cpumask

- [Overview](./cpumask/Overview.md)

- [API Reference](./cpumask/API_Reference.md)
    - [Construction and Conversion Methods](./cpumask/Construction_and_Conversion_Methods.md)
    - [Query and Inspection Operations](./cpumask/Query_and_Inspection_Operations.md)
    - [Modification and Iteration](./cpumask/Modification_and_Iteration.md)
    - [Bitwise Operations and Traits](./cpumask/Bitwise_Operations_and_Traits.md)

- [Architecture and Design](./cpumask/Architecture_and_Design.md)

- [Usage Guide and Examples](./cpumask/Usage_Guide_and_Examples.md)

- [Development and Contribution](./cpumask/Development_and_Contribution.md)

# axcpu

- [Overview](./axcpu/Overview.md)

- [x86_64 Architecture](./axcpu/x86_64_Architecture.md)
    - [x86_64 Context Management](./axcpu/x86_64_Context_Management.md)
    - [x86_64 Trap and Exception Handling](./axcpu/x86_64_Trap_and_Exception_Handling.md)
    - [x86_64 System Calls](./axcpu/x86_64_System_Calls.md)
    - [x86_64 System Initialization](./axcpu/x86_64_System_Initialization.md)

- [AArch64 Architecture](./axcpu/AArch64_Architecture.md)
    - [AArch64 Context Management](./axcpu/AArch64_Context_Management.md)
    - [AArch64 Trap and Exception Handling](./axcpu/AArch64_Trap_and_Exception_Handling.md)
    - [AArch64 System Initialization](./axcpu/AArch64_System_Initialization.md)

- [RISC-V Architecture](./axcpu/RISC-V_Architecture.md)
    - [RISC-V Context Management](./axcpu/RISC-V_Context_Management.md)
    - [RISC-V Trap and Exception Handling](./axcpu/RISC-V_Trap_and_Exception_Handling.md)
    - [RISC-V System Initialization](./axcpu/RISC-V_System_Initialization.md)

- [LoongArch64 Architecture](./axcpu/LoongArch64_Architecture.md)
    - [LoongArch64 Context Management](./axcpu/LoongArch64_Context_Management.md)
    - [LoongArch64 Assembly Operations](./axcpu/LoongArch64_Assembly_Operations.md)
    - [LoongArch64 System Initialization](./axcpu/LoongArch64_System_Initialization.md)

- [Cross-Architecture Features](./axcpu/Cross-Architecture_Features.md)
    - [User Space Support](./axcpu/User_Space_Support.md)
    - [Core Trap Handling Framework](./axcpu/Core_Trap_Handling_Framework.md)

- [Development and Build Configuration](./axcpu/Development_and_Build_Configuration.md)
    - [Dependencies and Package Structure](./axcpu/Dependencies_and_Package_Structure.md)
    - [Toolchain Configuration](./axcpu/Toolchain_Configuration.md)

# axfs_crates

- [Overview](./axfs_crates/Overview.md)
    - [Repository Structure](./axfs_crates/Repository_Structure.md)
    - [Development and Contribution](./axfs_crates/Development_and_Contribution.md)

- [File System Architecture](./axfs_crates/File_System_Architecture.md)
    - [Virtual File System Interface (axfs_vfs)](./axfs_crates/Virtual_File_System_Interface_(axfs_vfs).md)

- [Device File System (axfs_devfs)](./axfs_crates/Device_File_System_(axfs_devfs).md)
    - [Directory Structure](./axfs_crates/Directory_Structure.md)
    - [Null Device](./axfs_crates/Null_Device.md)
    - [Zero Device](./axfs_crates/Zero_Device.md)
    - [Usage Examples](./axfs_crates/Usage_Examples.md)

- [RAM File System (axfs_ramfs)](./axfs_crates/RAM_File_System_(axfs_ramfs).md)

# scheduler

- [Overview](./scheduler/Overview.md)

- [Core Architecture](./scheduler/Core_Architecture.md)

- [Scheduler Implementations](./scheduler/Scheduler_Implementations.md)
    - [FIFO Scheduler](./scheduler/FIFO_Scheduler.md)
    - [Completely Fair Scheduler (CFS)](./scheduler/Completely_Fair_Scheduler_(CFS).md)
    - [Round Robin Scheduler](./scheduler/Round_Robin_Scheduler.md)
    - [Scheduler Comparison](./scheduler/Scheduler_Comparison.md)

- [Testing Framework](./scheduler/Testing_Framework.md)

- [Development Guide](./scheduler/Development_Guide.md)

# tuple_for_each

- [Overview](./tuple_for_each/Overview.md)
    - [Project Structure](./tuple_for_each/Project_Structure.md)

- [Getting Started](./tuple_for_each/Getting_Started.md)
    - [Basic Usage](./tuple_for_each/Basic_Usage.md)
    - [Generated Functionality](./tuple_for_each/Generated_Functionality.md)

- [Implementation Guide](./tuple_for_each/Implementation_Guide.md)
    - [Derive Macro Processing](./tuple_for_each/Derive_Macro_Processing.md)
    - [Code Generation Pipeline](./tuple_for_each/Code_Generation_Pipeline.md)

- [Development](./tuple_for_each/Development.md)
    - [Testing](./tuple_for_each/Testing.md)
    - [CI/CD Pipeline](./tuple_for_each/CI_CD_Pipeline.md)

- [API Reference](./tuple_for_each/API_Reference.md)
    - [TupleForEach Derive Macro](./tuple_for_each/TupleForEach_Derive_Macro.md)
    - [Generated Macros](./tuple_for_each/Generated_Macros.md)
    - [Generated Methods](./tuple_for_each/Generated_Methods.md)

# allocator

- [Overview](./allocator/Overview.md)

- [Architecture and Design](./allocator/Architecture_and_Design.md)

- [Allocator Implementations](./allocator/Allocator_Implementations.md)
    - [Bitmap Page Allocator](./allocator/Bitmap_Page_Allocator.md)
    - [Buddy System Allocator](./allocator/Buddy_System_Allocator.md)
    - [Slab Allocator](./allocator/Slab_Allocator.md)
    - [TLSF Allocator](./allocator/TLSF_Allocator.md)

- [Usage and Configuration](./allocator/Usage_and_Configuration.md)

- [Testing and Benchmarks](./allocator/Testing_and_Benchmarks.md)
    - [Integration Tests](./allocator/Integration_Tests.md)
    - [Performance Benchmarks](./allocator/Performance_Benchmarks.md)

- [Development and Contributing](./allocator/Development_and_Contributing.md)

# axmm_crates

- [Overview](./axmm_crates/Overview.md)
    - [System Architecture](./axmm_crates/System_Architecture.md)

- [memory_addr Crate](./axmm_crates/memory_addr_Crate.md)
    - [Address Types and Operations](./axmm_crates/Address_Types_and_Operations.md)
    - [Address Ranges](./axmm_crates/Address_Ranges.md)
    - [Page Iteration](./axmm_crates/Page_Iteration.md)

- [memory_set Crate](./axmm_crates/memory_set_Crate.md)
    - [MemorySet Core](./axmm_crates/MemorySet_Core.md)
    - [MemoryArea](./axmm_crates/MemoryArea.md)
    - [MappingBackend](./axmm_crates/MappingBackend.md)
    - [Usage Examples and Testing](./axmm_crates/Usage_Examples_and_Testing.md)

- [Development Guide](./axmm_crates/Development_Guide.md)

# arm_pl031

- [Overview](./arm_pl031/Overview.md)
    - [Project Purpose and Scope](./arm_pl031/Project_Purpose_and_Scope.md)
    - [System Architecture](./arm_pl031/System_Architecture.md)

- [Getting Started](./arm_pl031/Getting_Started.md)
    - [Installation and Dependencies](./arm_pl031/Installation_and_Dependencies.md)
    - [Basic Usage and Examples](./arm_pl031/Basic_Usage_and_Examples.md)

- [Core Driver Implementation](./arm_pl031/Core_Driver_Implementation.md)
    - [Driver Architecture and Design](./arm_pl031/Driver_Architecture_and_Design.md)
    - [Hardware Interface and MMIO](./arm_pl031/Hardware_Interface_and_MMIO.md)
    - [Register Operations](./arm_pl031/Register_Operations.md)
    - [Interrupt Handling](./arm_pl031/Interrupt_Handling.md)
    - [Memory Safety and Concurrency](./arm_pl031/Memory_Safety_and_Concurrency.md)

- [Features and Extensions](./arm_pl031/Features_and_Extensions.md)
    - [Chrono Integration](./arm_pl031/Chrono_Integration.md)
    - [Feature Configuration](./arm_pl031/Feature_Configuration.md)

- [Development and Contributing](./arm_pl031/Development_and_Contributing.md)
    - [Building and Testing](./arm_pl031/Building_and_Testing.md)
    - [API Evolution and Changelog](./arm_pl031/API_Evolution_and_Changelog.md)
    - [Development Environment](./arm_pl031/Development_Environment.md)

# crate_interface

- [Overview](./crate_interface/Overview.md)

- [Getting Started](./crate_interface/Getting_Started.md)

- [Macro Reference](./crate_interface/Macro_Reference.md)
    - [def_interface Macro](./crate_interface/def_interface_Macro.md)
    - [impl_interface Macro](./crate_interface/impl_interface_Macro.md)
    - [call_interface Macro](./crate_interface/call_interface_Macro.md)

- [Architecture and Internals](./crate_interface/Architecture_and_Internals.md)

- [Development Guide](./crate_interface/Development_Guide.md)
    - [Testing](./crate_interface/Testing.md)
    - [CI/CD Pipeline](./crate_interface/CI_CD_Pipeline.md)
    - [Project Structure](./crate_interface/Project_Structure.md)

# lazyinit

- [Overview](./lazyinit/Overview.md)

- [LazyInit<T> Implementation](./lazyinit/LazyInit_T__Implementation.md)
    - [API Reference](./lazyinit/API_Reference.md)
    - [Thread Safety & Memory Model](./lazyinit/Thread_Safety_&_Memory_Model.md)
    - [Usage Patterns & Examples](./lazyinit/Usage_Patterns_&_Examples.md)

- [Project Configuration](./lazyinit/Project_Configuration.md)

- [Development & Contributing](./lazyinit/Development_&_Contributing.md)
    - [CI/CD Pipeline](./lazyinit/CI_CD_Pipeline.md)
    - [Development Environment Setup](./lazyinit/Development_Environment_Setup.md)

# linked_list_r4l

- [Overview](./linked_list_r4l/Overview.md)

- [Quick Start Guide](./linked_list_r4l/Quick_Start_Guide.md)

- [Architecture Overview](./linked_list_r4l/Architecture_Overview.md)

- [API Reference](./linked_list_r4l/API_Reference.md)
    - [User-Friendly API](./linked_list_r4l/User-Friendly_API.md)
    - [Advanced API](./linked_list_r4l/Advanced_API.md)
    - [Low-Level API](./linked_list_r4l/Low-Level_API.md)

- [Core Concepts](./linked_list_r4l/Core_Concepts.md)
    - [Memory Management](./linked_list_r4l/Memory_Management.md)
    - [Thread Safety](./linked_list_r4l/Thread_Safety.md)

- [Development Guide](./linked_list_r4l/Development_Guide.md)

# arm_pl011

- [Overview](./arm_pl011/Overview.md)
    - [Architecture](./arm_pl011/Architecture.md)
    - [Getting Started](./arm_pl011/Getting_Started.md)

- [Core Implementation](./arm_pl011/Core_Implementation.md)
    - [Register Definitions](./arm_pl011/Register_Definitions.md)
    - [UART Operations](./arm_pl011/UART_Operations.md)

- [API Reference](./arm_pl011/API_Reference.md)
    - [Pl011Uart Methods](./arm_pl011/Pl011Uart_Methods.md)
    - [Thread Safety and Memory Safety](./arm_pl011/Thread_Safety_and_Memory_Safety.md)

- [Development](./arm_pl011/Development.md)
    - [Building and Testing](./arm_pl011/Building_and_Testing.md)
    - [CI/CD Pipeline](./arm_pl011/CI_CD_Pipeline.md)

- [Hardware Reference](./arm_pl011/Hardware_Reference.md)

# handler_table

- [Introduction](./handler_table/Introduction.md)
    - [ArceOS Integration](./handler_table/ArceOS_Integration.md)
    - [Lock-free Design Benefits](./handler_table/Lock-free_Design_Benefits.md)

- [User Guide](./handler_table/User_Guide.md)
    - [API Reference](./handler_table/API_Reference.md)
    - [Usage Examples](./handler_table/Usage_Examples.md)

- [Implementation Details](./handler_table/Implementation_Details.md)
    - [Atomic Operations](./handler_table/Atomic_Operations.md)
    - [Memory Layout and Safety](./handler_table/Memory_Layout_and_Safety.md)

- [Development Guide](./handler_table/Development_Guide.md)
    - [Building and Testing](./handler_table/Building_and_Testing.md)
    - [CI/CD Pipeline](./handler_table/CI_CD_Pipeline.md)

# memory_set

- [Overview](./memory_set/Overview.md)
    - [Core Concepts](./memory_set/Core_Concepts.md)
    - [System Architecture](./memory_set/System_Architecture.md)

- [Implementation Details](./memory_set/Implementation_Details.md)
    - [MemoryArea and MappingBackend](./memory_set/MemoryArea_and_MappingBackend.md)
    - [MemorySet Collection Management](./memory_set/MemorySet_Collection_Management.md)
    - [Public API and Error Handling](./memory_set/Public_API_and_Error_Handling.md)

- [Usage and Examples](./memory_set/Usage_and_Examples.md)
    - [Basic Usage Patterns](./memory_set/Basic_Usage_Patterns.md)
    - [Advanced Examples and Testing](./memory_set/Advanced_Examples_and_Testing.md)

- [Development and Project Setup](./memory_set/Development_and_Project_Setup.md)
    - [Dependencies and Configuration](./memory_set/Dependencies_and_Configuration.md)
    - [Development Workflow](./memory_set/Development_Workflow.md)

# int_ratio

- [Overview](./int_ratio/Overview.md)

- [Ratio Type Implementation](./int_ratio/Ratio_Type_Implementation.md)
    - [Internal Architecture](./int_ratio/Internal_Architecture.md)
    - [API Reference](./int_ratio/API_Reference.md)
    - [Mathematical Foundation](./int_ratio/Mathematical_Foundation.md)

- [Usage Guide](./int_ratio/Usage_Guide.md)

- [Development Guide](./int_ratio/Development_Guide.md)

# flatten_objects

- [Overview](./flatten_objects/Overview.md)
    - [Project Structure and Dependencies](./flatten_objects/Project_Structure_and_Dependencies.md)
    - [Key Concepts and Terminology](./flatten_objects/Key_Concepts_and_Terminology.md)

- [FlattenObjects API Documentation](./flatten_objects/FlattenObjects_API_Documentation.md)
    - [Container Creation and Configuration](./flatten_objects/Container_Creation_and_Configuration.md)
    - [Object Management Operations](./flatten_objects/Object_Management_Operations.md)
    - [Query and Inspection Methods](./flatten_objects/Query_and_Inspection_Methods.md)

- [Implementation Details](./flatten_objects/Implementation_Details.md)
    - [Internal Data Structures](./flatten_objects/Internal_Data_Structures.md)
    - [Memory Management and Safety](./flatten_objects/Memory_Management_and_Safety.md)
    - [ID Management System](./flatten_objects/ID_Management_System.md)

- [Usage Guide and Examples](./flatten_objects/Usage_Guide_and_Examples.md)
    - [Basic Operations](./flatten_objects/Basic_Operations.md)
    - [Advanced Patterns and Best Practices](./flatten_objects/Advanced_Patterns_and_Best_Practices.md)

- [Development and Maintenance](./flatten_objects/Development_and_Maintenance.md)
    - [Building and Testing](./flatten_objects/Building_and_Testing.md)
    - [Project Configuration](./flatten_objects/Project_Configuration.md)

# arm_gicv2

- [Overview](./arm_gicv2/Overview.md)

- [Interrupt System Architecture](./arm_gicv2/Interrupt_System_Architecture.md)
    - [Interrupt Types and Ranges](./arm_gicv2/Interrupt_Types_and_Ranges.md)
    - [Trigger Modes and Translation](./arm_gicv2/Trigger_Modes_and_Translation.md)

- [Hardware Interface Implementation](./arm_gicv2/Hardware_Interface_Implementation.md)
    - [GIC Distributor](./arm_gicv2/GIC_Distributor.md)
    - [GIC CPU Interface](./arm_gicv2/GIC_CPU_Interface.md)

- [Development Guide](./arm_gicv2/Development_Guide.md)
    - [Build System and Dependencies](./arm_gicv2/Build_System_and_Dependencies.md)
    - [CI/CD Pipeline](./arm_gicv2/CI_CD_Pipeline.md)
    - [Development Environment](./arm_gicv2/Development_Environment.md)

# axconfig-gen

- [Overview](./axconfig-gen/Overview.md)
    - [System Architecture](./axconfig-gen/System_Architecture.md)
    - [Quick Start Guide](./axconfig-gen/Quick_Start_Guide.md)

- [axconfig-gen Package](./axconfig-gen/axconfig-gen_Package.md)
    - [Command Line Interface](./axconfig-gen/Command_Line_Interface.md)
    - [Library API](./axconfig-gen/Library_API.md)
        - [Core Data Structures](./axconfig-gen/Core_Data_Structures.md)
        - [Type System](./axconfig-gen/Type_System.md)
        - [Output Generation](./axconfig-gen/Output_Generation.md)

- [axconfig-macros Package](./axconfig-gen/axconfig-macros_Package.md)
    - [Macro Usage Patterns](./axconfig-gen/Macro_Usage_Patterns.md)
    - [Macro Implementation](./axconfig-gen/Macro_Implementation.md)

- [Configuration Examples](./axconfig-gen/Configuration_Examples.md)
    - [TOML Configuration Format](./axconfig-gen/TOML_Configuration_Format.md)
    - [Generated Output Examples](./axconfig-gen/Generated_Output_Examples.md)

- [Development Guide](./axconfig-gen/Development_Guide.md)
    - [Build System and Dependencies](./axconfig-gen/Build_System_and_Dependencies.md)
    - [Testing](./axconfig-gen/Testing.md)
    - [Continuous Integration](./axconfig-gen/Continuous_Integration.md)

# page_table_multiarch

- [Overview](./page_table_multiarch/Overview.md)
    - [System Architecture](./page_table_multiarch/System_Architecture.md)
    - [Supported Platforms](./page_table_multiarch/Supported_Platforms.md)

- [Workspace Structure](./page_table_multiarch/Workspace_Structure.md)
    - [page_table_multiarch Crate](./page_table_multiarch/page_table_multiarch_Crate.md)
    - [page_table_entry Crate](./page_table_multiarch/page_table_entry_Crate.md)

- [Core Concepts](./page_table_multiarch/Core_Concepts.md)
    - [PageTable64 Implementation](./page_table_multiarch/PageTable64_Implementation.md)
    - [Generic Traits System](./page_table_multiarch/Generic_Traits_System.md)
    - [Memory Mapping Flags](./page_table_multiarch/Memory_Mapping_Flags.md)

- [Architecture Support](./page_table_multiarch/Architecture_Support.md)
    - [x86_64 Support](./page_table_multiarch/x86_64_Support.md)
    - [AArch64 Support](./page_table_multiarch/AArch64_Support.md)
    - [RISC-V Support](./page_table_multiarch/RISC-V_Support.md)
    - [LoongArch64 Support](./page_table_multiarch/LoongArch64_Support.md)

- [Development Guide](./page_table_multiarch/Development_Guide.md)
    - [Building and Testing](./page_table_multiarch/Building_and_Testing.md)
    - [Contributing](./page_table_multiarch/Contributing.md)

# x86_rtc

- [Overview](./x86_rtc/Overview.md)
    - [Crate Definition and Metadata](./x86_rtc/Crate_Definition_and_Metadata.md)
    - [Quick Start Guide](./x86_rtc/Quick_Start_Guide.md)

- [Implementation](./x86_rtc/Implementation.md)
    - [RTC Driver API](./x86_rtc/RTC_Driver_API.md)
    - [CMOS Hardware Interface](./x86_rtc/CMOS_Hardware_Interface.md)
    - [Data Format Handling](./x86_rtc/Data_Format_Handling.md)

- [Dependencies and Platform Support](./x86_rtc/Dependencies_and_Platform_Support.md)
    - [Dependency Analysis](./x86_rtc/Dependency_Analysis.md)
    - [Platform and Architecture Requirements](./x86_rtc/Platform_and_Architecture_Requirements.md)

- [Development Workflow](./x86_rtc/Development_Workflow.md)
    - [CI/CD Pipeline](./x86_rtc/CI_CD_Pipeline.md)
    - [Development Environment Setup](./x86_rtc/Development_Environment_Setup.md)

# percpu

- [Overview](./percpu/Overview.md)
    - [System Components](./percpu/System_Components.md)
    - [Supported Platforms](./percpu/Supported_Platforms.md)

- [Getting Started](./percpu/Getting_Started.md)
    - [Installation and Setup](./percpu/Installation_and_Setup.md)
    - [Basic Usage Examples](./percpu/Basic_Usage_Examples.md)
    - [Feature Flags Configuration](./percpu/Feature_Flags_Configuration.md)

- [Architecture and Design](./percpu/Architecture_and_Design.md)
    - [Memory Layout and Initialization](./percpu/Memory_Layout_and_Initialization.md)
    - [Cross-Platform Abstraction](./percpu/Cross-Platform_Abstraction.md)
    - [Code Generation Pipeline](./percpu/Code_Generation_Pipeline.md)

- [API Reference](./percpu/API_Reference.md)
    - [def_percpu Macro](./percpu/def_percpu_Macro.md)
    - [Runtime Functions](./percpu/Runtime_Functions.md)
    - [Safety and Preemption](./percpu/Safety_and_Preemption.md)

- [Implementation Details](./percpu/Implementation_Details.md)
    - [Architecture-Specific Code Generation](./percpu/Architecture-Specific_Code_Generation.md)
    - [Naive Implementation](./percpu/Naive_Implementation.md)
    - [Memory Management Internals](./percpu/Memory_Management_Internals.md)

- [Development and Testing](./percpu/Development_and_Testing.md)
    - [Testing Guide](./percpu/Testing_Guide.md)
    - [Build System](./percpu/Build_System.md)
    - [Contributing](./percpu/Contributing.md)

# axdriver_crates

- [Overview](./axdriver_crates/Overview.md)

- [Architecture and Design](./axdriver_crates/Architecture_and_Design.md)

- [Foundation Layer (axdriver_base)](./axdriver_crates/Foundation_Layer_(axdriver_base).md)

- [Network Drivers](./axdriver_crates/Network_Drivers.md)
    - [Network Driver Interface](./axdriver_crates/Network_Driver_Interface.md)
    - [Network Buffer Management](./axdriver_crates/Network_Buffer_Management.md)
    - [Hardware Implementations](./axdriver_crates/Hardware_Implementations.md)

- [Block Storage Drivers](./axdriver_crates/Block_Storage_Drivers.md)
    - [Block Driver Interface](./axdriver_crates/Block_Driver_Interface.md)
    - [Block Device Implementations](./axdriver_crates/Block_Device_Implementations.md)

- [Display Drivers](./axdriver_crates/Display_Drivers.md)

- [VirtIO Integration](./axdriver_crates/VirtIO_Integration.md)
    - [VirtIO Core Abstraction](./axdriver_crates/VirtIO_Core_Abstraction.md)
    - [VirtIO Device Implementations](./axdriver_crates/VirtIO_Device_Implementations.md)

- [PCI Bus Operations](./axdriver_crates/PCI_Bus_Operations.md)

- [Development and Build Configuration](./axdriver_crates/Development_and_Build_Configuration.md)

# axio

- [Overview](./axio/Overview.md)

- [Core I/O Traits](./axio/Core_I_O_Traits.md)

- [Crate Configuration and Features](./axio/Crate_Configuration_and_Features.md)

- [Implementations](./axio/Implementations.md)
    - [Buffered I/O](./axio/Buffered_I_O.md)
    - [Basic Type Implementations](./axio/Basic_Type_Implementations.md)

- [Supporting Systems](./axio/Supporting_Systems.md)
    - [Error Handling](./axio/Error_Handling.md)
    - [Prelude Module](./axio/Prelude_Module.md)

- [Development and Maintenance](./axio/Development_and_Maintenance.md)
    - [Build System and CI](./axio/Build_System_and_CI.md)
    - [Project Configuration](./axio/Project_Configuration.md)

# riscv_goldfish

- [Overview](./riscv_goldfish/Overview.md)
    - [Architecture Overview](./riscv_goldfish/Architecture_Overview.md)

- [RTC Driver Implementation](./riscv_goldfish/RTC_Driver_Implementation.md)
    - [API Reference](./riscv_goldfish/API_Reference.md)
    - [Hardware Interface](./riscv_goldfish/Hardware_Interface.md)
    - [Time Conversion](./riscv_goldfish/Time_Conversion.md)

- [Project Configuration](./riscv_goldfish/Project_Configuration.md)
    - [Target Platforms and Cross-Compilation](./riscv_goldfish/Target_Platforms_and_Cross-Compilation.md)
    - [Licensing and Distribution](./riscv_goldfish/Licensing_and_Distribution.md)

- [Development Workflow](./riscv_goldfish/Development_Workflow.md)
    - [CI/CD Pipeline](./riscv_goldfish/CI_CD_Pipeline.md)
    - [Development Environment Setup](./riscv_goldfish/Development_Environment_Setup.md)
