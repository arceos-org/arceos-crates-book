# LoongArch64 Assembly Operations

> **Relevant source files**
> * [src/aarch64/asm.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/aarch64/asm.rs)
> * [src/loongarch64/asm.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/asm.rs)
> * [src/loongarch64/macros.rs](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/macros.rs)

This document covers the low-level assembly operations and macros provided by the LoongArch64 architecture implementation in axcpu. It focuses on the assembly-level primitives used for CPU state management, register manipulation, and hardware control operations.

For information about LoongArch64 context structures and high-level context switching, see [LoongArch64 Context Management](/arceos-org/axcpu/5.1-loongarch64-context-management). For system initialization procedures, see [LoongArch64 System Initialization](/arceos-org/axcpu/5.3-loongarch64-system-initialization).

## Control and Status Register (CSR) Operations

The LoongArch64 implementation defines a comprehensive set of Control and Status Register (CSR) constants and operations. These registers control fundamental CPU behaviors including exception handling, memory management, and system configuration.

### CSR Definitions

The core CSR registers are defined as assembly constants for direct hardware access:

|Register|Value|Purpose|
| --- | --- | --- |
|LA_CSR_PRMD|0x1|Previous Mode Data - saves processor state|
|LA_CSR_EUEN|0x2|Extended Unit Enable - controls extensions|
|LA_CSR_ERA|0x6|Exception Return Address|
|LA_CSR_PGDL|0x19|Page table base when VA[47] = 0|
|LA_CSR_PGDH|0x1a|Page table base when VA[47] = 1|
|LA_CSR_PGD|0x1b|General page table base|
|LA_CSR_TLBRENTRY|0x88|TLB refill exception entry|
|LA_CSR_DMW0/DMW1|0x180/0x181|Direct Mapped Windows|

```mermaid
flowchart TD
subgraph CSR_SYSTEM["LoongArch64 CSR System"]
    subgraph KSAVE_REGS["Kernel Save Registers"]
        KSAVE_KSP["KSAVE_KSPKernel Stack Pointer"]
        KSAVE_TEMP["KSAVE_TEMPTemporary Storage"]
        KSAVE_R21["KSAVE_R21Register 21 Save"]
        KSAVE_TP["KSAVE_TPThread Pointer Save"]
        EUEN["LA_CSR_EUENExtended Unit Enable"]
        TLBRENTRY["LA_CSR_TLBRENTRYTLB Refill Entry"]
        PGDL["LA_CSR_PGDLUser Page Table"]
        PRMD["LA_CSR_PRMDPrevious Mode Data"]
    end
    subgraph EXTENSION_CSRS["Extensions"]
        subgraph TLB_CSRS["TLB Management"]
            KSAVE_KSP["KSAVE_KSPKernel Stack Pointer"]
            EUEN["LA_CSR_EUENExtended Unit Enable"]
            DMW0["LA_CSR_DMW0Direct Map Window 0"]
            DMW1["LA_CSR_DMW1Direct Map Window 1"]
            TLBRENTRY["LA_CSR_TLBRENTRYTLB Refill Entry"]
            TLBRBADV["LA_CSR_TLBRBADVTLB Bad Address"]
            TLBRERA["LA_CSR_TLBRERATLB Refill ERA"]
            PGDL["LA_CSR_PGDLUser Page Table"]
            PGDH["LA_CSR_PGDHKernel Page Table"]
            PWCL["LA_CSR_PWCLPage Walk Lower"]
            PRMD["LA_CSR_PRMDPrevious Mode Data"]
            ERA["LA_CSR_ERAException Return Address"]
            EENTRY["Exception Entry Base"]
        end
    end
    subgraph MEMORY_CSRS["Memory Management"]
        KSAVE_KSP["KSAVE_KSPKernel Stack Pointer"]
        EUEN["LA_CSR_EUENExtended Unit Enable"]
        DMW0["LA_CSR_DMW0Direct Map Window 0"]
        DMW1["LA_CSR_DMW1Direct Map Window 1"]
        TLBRENTRY["LA_CSR_TLBRENTRYTLB Refill Entry"]
        TLBRBADV["LA_CSR_TLBRBADVTLB Bad Address"]
        TLBRERA["LA_CSR_TLBRERATLB Refill ERA"]
        PGDL["LA_CSR_PGDLUser Page Table"]
        PGDH["LA_CSR_PGDHKernel Page Table"]
        PWCL["LA_CSR_PWCLPage Walk Lower"]
        PWCH["LA_CSR_PWCHPage Walk Higher"]
        PRMD["LA_CSR_PRMDPrevious Mode Data"]
        ERA["LA_CSR_ERAException Return Address"]
        EENTRY["Exception Entry Base"]
    end
    subgraph EXCEPTION_CSRS["Exception Control"]
        KSAVE_KSP["KSAVE_KSPKernel Stack Pointer"]
        EUEN["LA_CSR_EUENExtended Unit Enable"]
        DMW0["LA_CSR_DMW0Direct Map Window 0"]
        DMW1["LA_CSR_DMW1Direct Map Window 1"]
        TLBRENTRY["LA_CSR_TLBRENTRYTLB Refill Entry"]
        TLBRBADV["LA_CSR_TLBRBADVTLB Bad Address"]
        TLBRERA["LA_CSR_TLBRERATLB Refill ERA"]
        PGDL["LA_CSR_PGDLUser Page Table"]
        PGDH["LA_CSR_PGDHKernel Page Table"]
        PWCL["LA_CSR_PWCLPage Walk Lower"]
        PRMD["LA_CSR_PRMDPrevious Mode Data"]
        ERA["LA_CSR_ERAException Return Address"]
        EENTRY["Exception Entry Base"]
    end
end
```

Sources: [src/loongarch64/macros.rs(L7 - L29)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/macros.rs#L7-L29)

### Register Access Macros

The implementation provides optimized macros for common register operations:

* `STD rd, rj, off` - Store doubleword with scaled offset
* `LDD rd, rj, off` - Load doubleword with scaled offset

These macros automatically scale the offset by 8 bytes for 64-bit operations, simplifying stack and structure access patterns.

Sources: [src/loongarch64/macros.rs(L31 - L36)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/macros.rs#L31-L36)

## General Purpose Register Operations

### Register Save/Restore Framework

The LoongArch64 implementation provides a systematic approach to saving and restoring general purpose registers using parameterized macros:

```mermaid
flowchart TD
subgraph GPR_OPERATIONS["General Purpose Register Operations"]
    subgraph CONCRETE_MACROS["Concrete Operations"]
        PUSH_GENERAL["PUSH_GENERAL_REGSSave All GPRs"]
        POP_GENERAL["POP_GENERAL_REGSRestore All GPRs"]
    end
    subgraph REGISTER_GROUPS["Register Groups"]
        ARGS["Argument Registersa0-a7"]
        TEMPS["Temporary Registerst0-t8"]
        SAVED["Saved Registerss0-s8"]
        SPECIAL["Special Registersra, fp, sp"]
    end
    subgraph MACRO_FRAMEWORK["Macro Framework"]
        PUSH_POP["PUSH_POP_GENERAL_REGSParameterized Operation"]
        STD_OP["STD OperationStore Doubleword"]
        LDD_OP["LDD OperationLoad Doubleword"]
    end
end

ARGS --> PUSH_GENERAL
POP_GENERAL --> PUSH_POP
PUSH_GENERAL --> PUSH_POP
PUSH_POP --> LDD_OP
PUSH_POP --> STD_OP
SAVED --> PUSH_GENERAL
SPECIAL --> PUSH_GENERAL
TEMPS --> PUSH_GENERAL
```

The `PUSH_POP_GENERAL_REGS` macro takes an operation parameter (`STD` for save, `LDD` for restore) and systematically processes all general purpose registers in a standardized order. This ensures consistent register handling across all context switching operations.

Sources: [src/loongarch64/macros.rs(L38 - L74)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/macros.rs#L38-L74)

### Register Layout and Ordering

The register save/restore operations follow the LoongArch64 ABI conventions:

|Offset|Register|Purpose|
| --- | --- | --- |
|1|$ra|Return address|
|4-11|$a0-$a7|Function arguments|
|12-20|$t0-$t8|Temporary registers|
|22|$fp|Frame pointer|
|23-31|$s0-$s8|Saved registers|

Sources: [src/loongarch64/macros.rs(L39 - L66)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/macros.rs#L39-L66)

## Floating Point Operations

When the `fp-simd` feature is enabled, the LoongArch64 implementation provides comprehensive floating point state management through specialized assembly macros.

### Floating Point Condition Code (FCC) Management

The FCC registers (`fcc0`-`fcc7`) store comparison results and require special handling due to their packed storage format:

```mermaid
flowchart TD
subgraph FCC_OPERATIONS["Floating Point Condition Code Operations"]
    FCC_UNPACK["Unpack bit fieldsbstrpick.d instructions"]
    FCC_WRITE["Write fcc0-fcc7movgr2cf instructions"]
    FCC_PACK["Pack into single registerbstrins.d instructions"]
    FCC_STORE["Store to memoryst.d instruction"]
    subgraph BIT_LAYOUT["64-bit FCC Layout"]
        FCC0_BITS["fcc0: bits 7:0"]
        FCC1_BITS["fcc1: bits 15:8"]
        FCC2_BITS["fcc2: bits 23:16"]
        FCC3_BITS["fcc3: bits 31:24"]
        FCC4_BITS["fcc4: bits 39:32"]
        FCC5_BITS["fcc5: bits 47:40"]
        FCC6_BITS["fcc6: bits 55:48"]
        FCC7_BITS["fcc7: bits 63:56"]
        FCC_LOAD["Load from memoryld.d instruction"]
        FCC_READ["Read fcc0-fcc7movcf2gr instructions"]
    end
    subgraph FCC_RESTORE["RESTORE_FCC Macro Flow"]
        subgraph FCC_SAVE["SAVE_FCC Macro Flow"]
            FCC0_BITS["fcc0: bits 7:0"]
            FCC_LOAD["Load from memoryld.d instruction"]
            FCC_UNPACK["Unpack bit fieldsbstrpick.d instructions"]
            FCC_WRITE["Write fcc0-fcc7movgr2cf instructions"]
            FCC_READ["Read fcc0-fcc7movcf2gr instructions"]
            FCC_PACK["Pack into single registerbstrins.d instructions"]
            FCC_STORE["Store to memoryst.d instruction"]
        end
    end
end

FCC_LOAD --> FCC_UNPACK
FCC_PACK --> FCC_STORE
FCC_READ --> FCC_PACK
FCC_UNPACK --> FCC_WRITE
```

The implementation uses bit manipulation instructions (`bstrins.d`, `bstrpick.d`) to efficiently pack and unpack the 8 condition code registers into a single 64-bit value for storage.

Sources: [src/loongarch64/macros.rs(L87 - L125)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/macros.rs#L87-L125)

### Floating Point Control and Status Register (FCSR)

The FCSR management is simpler than FCC handling since it's a single 32-bit register:

* `SAVE_FCSR` - Uses `movfcsr2gr` to read FCSR and `st.w` to store
* `RESTORE_FCSR` - Uses `ld.w` to load and `movgr2fcsr` to write back

Sources: [src/loongarch64/macros.rs(L127 - L135)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/macros.rs#L127-L135)

### Floating Point Register Operations

The floating point registers (`$f0`-`$f31`) are handled through a parameterized macro system similar to general purpose registers:

```mermaid
flowchart TD
subgraph FP_REG_OPS["Floating Point Register Operations"]
    subgraph FP_REGISTERS["32 Floating Point Registers"]
        F0_F7["f0-f7Temporary registers"]
        F8_F15["f8-f15Temporary registers"]
        F16_F23["f16-f23Saved registers"]
        F24_F31["f24-f31Saved registers"]
    end
    subgraph FP_CONCRETE["Concrete Operations"]
        SAVE_FP["SAVE_FP MacroUses fst.d operation"]
        RESTORE_FP["RESTORE_FP MacroUses fld.d operation"]
    end
    subgraph FP_MACRO["PUSH_POP_FLOAT_REGS Macro"]
        FP_PARAM["Takes operation and base registerfst.d or fld.d"]
        FP_LOOP["Iterates f0 through f31Sequential 8-byte offsets"]
    end
end

F0_F7 --> FP_LOOP
F16_F23 --> FP_LOOP
F24_F31 --> FP_LOOP
F8_F15 --> FP_LOOP
FP_PARAM --> RESTORE_FP
FP_PARAM --> SAVE_FP
```

Sources: [src/loongarch64/macros.rs(L138 - L179)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/macros.rs#L138-L179)

## Memory Management Operations

The LoongArch64 architecture provides sophisticated memory management capabilities through dedicated wrapper functions that interact with hardware registers and TLB operations.

### Page Table Management

LoongArch64 uses separate page table base registers for user and kernel address spaces:

|Function|Register|Address Space|
| --- | --- | --- |
|read_user_page_table()|PGDL|VA[47] = 0 (user space)|
|read_kernel_page_table()|PGDH|VA[47] = 1 (kernel space)|
|write_user_page_table()|PGDL|User space page table root|
|write_kernel_page_table()|PGDH|Kernel space page table root|

```mermaid
flowchart TD
subgraph PAGE_TABLE_OPS["Page Table Operations"]
    subgraph ADDRESS_SPACES["Virtual Address Spaces"]
        USER_SPACE["User SpaceVA[47] = 0"]
        KERNEL_SPACE["Kernel SpaceVA[47] = 1"]
    end
    subgraph HARDWARE_REGS["Hardware Registers"]
        PGDL_REG["PGDL RegisterUser Space Page Table"]
        PGDH_REG["PGDH RegisterKernel Space Page Table"]
    end
    subgraph WRITE_OPS["Write Operations"]
        WRITE_USER["write_user_page_table()pgdl::set_base()"]
        WRITE_KERNEL["write_kernel_page_table()pgdh::set_base()"]
    end
    subgraph READ_OPS["Read Operations"]
        READ_USER["read_user_page_table()pgdl::read().base()"]
        READ_KERNEL["read_kernel_page_table()pgdh::read().base()"]
    end
end

PGDH_REG --> KERNEL_SPACE
PGDL_REG --> USER_SPACE
READ_KERNEL --> PGDH_REG
READ_USER --> PGDL_REG
WRITE_KERNEL --> PGDH_REG
WRITE_USER --> PGDL_REG
```

Sources: [src/loongarch64/asm.rs(L41 - L79)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/asm.rs#L41-L79)

### TLB Management

The Translation Lookaside Buffer (TLB) management uses the `invtlb` instruction with specific operation codes:

* `invtlb 0x00` - Clear all TLB entries (full flush)
* `invtlb 0x05` - Clear specific VA with ASID=0 (single page flush)

The implementation includes proper memory barriers (`dbar 0`) to ensure ordering of memory operations around TLB invalidation.

Sources: [src/loongarch64/asm.rs(L81 - L111)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/asm.rs#L81-L111)

### Page Walk Controller Configuration

The `write_pwc()` function configures the hardware page walker through the PWCL and PWCH registers, which control page table traversal parameters for lower and upper address spaces respectively.

Sources: [src/loongarch64/asm.rs(L130 - L150)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/asm.rs#L130-L150)

## Exception and Interrupt Handling

### Interrupt Control

The LoongArch64 interrupt system is controlled through the Current Mode register (CRMD):

|Function|Operation|Hardware Effect|
| --- | --- | --- |
|enable_irqs()|crmd::set_ie(true)|Sets IE bit in CRMD|
|disable_irqs()|crmd::set_ie(false)|Clears IE bit in CRMD|
|irqs_enabled()|crmd::read().ie()|Reads IE bit status|

```mermaid
flowchart TD
subgraph IRQ_CONTROL["Interrupt Control System"]
    subgraph HARDWARE_STATE["Hardware State"]
        CRMD_REG["CRMD RegisterCurrent Mode Data"]
        IE_BIT["IE BitInterrupt Enable"]
        CPU_STATE["CPU Execution StateRunning/Idle"]
    end
    subgraph POWER_MANAGEMENT["Power Management"]
        WAIT_IRQS["wait_for_irqs()loongArch64::asm::idle()"]
        HALT_CPU["halt()disable_irqs() + idle()"]
    end
    subgraph IRQ_FUNCTIONS["IRQ Control Functions"]
        ENABLE_IRQS["enable_irqs()crmd::set_ie(true)"]
        DISABLE_IRQS["disable_irqs()crmd::set_ie(false)"]
        CHECK_IRQS["irqs_enabled()crmd::read().ie()"]
    end
end

CHECK_IRQS --> IE_BIT
DISABLE_IRQS --> IE_BIT
ENABLE_IRQS --> IE_BIT
HALT_CPU --> CPU_STATE
HALT_CPU --> DISABLE_IRQS
IE_BIT --> CRMD_REG
WAIT_IRQS --> CPU_STATE
```

Sources: [src/loongarch64/asm.rs(L8 - L39)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/asm.rs#L8-L39)

### Exception Entry Configuration

The `write_exception_entry_base()` function configures the exception handling entry point by setting both the Exception Entry Base Address register (EENTRY) and the Exception Configuration register (ECFG):

* Sets `ECFG.VS = 0` for unified exception entry
* Sets `EENTRY` to the provided exception handler address

Sources: [src/loongarch64/asm.rs(L113 - L128)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/asm.rs#L113-L128)

## Thread Local Storage Support

LoongArch64 implements Thread Local Storage (TLS) through the thread pointer register (`$tp`):

* `read_thread_pointer()` - Reads current `$tp` value using `move` instruction
* `write_thread_pointer()` - Updates `$tp` register for TLS base address

The implementation also provides kernel stack pointer management through a custom CSR (`KSAVE_KSP`) when the `uspace` feature is enabled, supporting user-space context switching.

Sources: [src/loongarch64/asm.rs(L152 - L199)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/asm.rs#L152-L199)

## Extension Support

### Floating Point Extensions

The LoongArch64 implementation supports multiple floating point and SIMD extensions:

* `enable_fp()` - Enables floating-point instructions by setting `EUEN.FPE`
* `enable_lsx()` - Enables LSX (LoongArch SIMD eXtension) by setting `EUEN.SXE`

These functions control the Extended Unit Enable register (EUEN) to selectively enable hardware extensions based on system requirements.

Sources: [src/loongarch64/asm.rs(L174 - L187)&emsp;](https://github.com/arceos-org/axcpu/blob/b93d8fa3/src/loongarch64/asm.rs#L174-L187)