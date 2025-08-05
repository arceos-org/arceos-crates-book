# LoongArch64 Support

> **Relevant source files**
> * [page_table_entry/src/arch/loongarch64.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/loongarch64.rs)
> * [page_table_multiarch/src/arch/loongarch64.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/loongarch64.rs)

This page documents the LoongArch64 architecture support within the `page_table_multiarch` library. LoongArch64 is implemented as one of the four supported architectures, providing 4-level page table management with architecture-specific features like Page Walk Controllers (PWC) and privilege level control.

For information about the overall multi-architecture design, see [Architecture Support](/arceos-org/page_table_multiarch/4-architecture-support). For implementation details of other architectures, see [x86_64 Support](/arceos-org/page_table_multiarch/4.1-x86_64-support), [AArch64 Support](/arceos-org/page_table_multiarch/4.2-aarch64-support), and [RISC-V Support](/arceos-org/page_table_multiarch/4.3-risc-v-support).

## Architecture Overview

LoongArch64 uses a 4-level page table structure with 48-bit physical and virtual addresses. The implementation leverages LoongArch64's Page Walk Controller (PWC) mechanism for hardware-assisted page table walking and provides comprehensive memory management capabilities including privilege level control and memory access type configuration.

### Page Table Structure

LoongArch64 employs a 4-level hierarchical page table structure consisting of DIR3, DIR2, DIR1, and PT levels. The DIR4 level is ignored in this implementation, focusing on the standard 4-level configuration.

```mermaid
flowchart TD
subgraph subGraph2["PWC Configuration"]
    PWCL["PWCL CSRLower Half Address SpacePTBase=12, PTWidth=9Dir1Base=21, Dir1Width=9Dir2Base=30, Dir2Width=9"]
    PWCH["PWCH CSRHigher Half Address SpaceDir3Base=39, Dir3Width=9"]
end
subgraph subGraph0["LoongArch64 4-Level Page Table"]
    VA["Virtual Address (48-bit)"]
    DIR3["DIR3 Table (Level 3)"]
    DIR2["DIR2 Table (Level 2)"]
    DIR1["DIR1 Table (Level 1)"]
    PT["Page Table (Level 0)"]
    PA["Physical Address (48-bit)"]
end

DIR1 --> PT
DIR2 --> DIR1
DIR3 --> DIR2
PT --> PA
PWCH --> DIR3
PWCL --> DIR1
PWCL --> DIR2
PWCL --> PT
VA --> DIR3
```

Sources: [page_table_multiarch/src/arch/loongarch64.rs(L11 - L41)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/loongarch64.rs#L11-L41) [page_table_multiarch/src/arch/loongarch64.rs(L43 - L46)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/loongarch64.rs#L43-L46)

## Page Table Entry Implementation

The LoongArch64 page table entry is implemented through the `LA64PTE` struct, which provides a complete set of architecture-specific flags and integrates with the generic trait system.

### LA64PTE Structure

The `LA64PTE` struct wraps a 64-bit value containing both the physical address and control flags. The physical address occupies bits 12-47, providing support for 48-bit physical addressing.

```mermaid
flowchart TD
subgraph subGraph0["LA64PTE Bit Layout (64 bits)"]
    RPLV["RPLV (63)Privilege Level Restricted"]
    NX["NX (62)Not Executable"]
    NR["NR (61)Not Readable"]
    UNUSED["Unused (60-13)"]
    G["G (12)Global (Huge Page)"]
    ADDR["Physical Address (47-12)36 bits"]
    W["W (8)Writable"]
    P["P (7)Present"]
    GH["GH (6)Global/Huge"]
    MATH["MATH (5)Memory Access Type High"]
    MATL["MATL (4)Memory Access Type Low"]
    PLVH["PLVH (3)Privilege Level High"]
    PLVL["PLVL (2)Privilege Level Low"]
    D["D (1)Dirty"]
    V["V (0)Valid"]
end
```

Sources: [page_table_entry/src/arch/loongarch64.rs(L11 - L59)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/loongarch64.rs#L11-L59) [page_table_entry/src/arch/loongarch64.rs(L121 - L133)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/loongarch64.rs#L121-L133)

### LoongArch64-Specific Flags

The `PTEFlags` bitflags define comprehensive control over memory access and privilege levels:

|Flag|Bit|Purpose|
| --- | --- | --- |
|V|0|Page table entry is valid|
|D|1|Page has been written (dirty bit)|
|PLVL/PLVH|2-3|Privilege level control (4 levels)|
|MATL/MATH|4-5|Memory Access Type (SUC/CC/WUC)|
|GH|6|Global mapping or huge page indicator|
|P|7|Physical page exists|
|W|8|Page is writable|
|G|12|Global mapping for huge pages|
|NR|61|Page is not readable|
|NX|62|Page is not executable|
|RPLV|63|Privilege level restriction|

The Memory Access Type (MAT) field supports three modes:

* `00` (SUC): Strongly-ordered uncached
* `01` (CC): Coherent cached
* `10` (WUC): Weakly-ordered uncached
* `11`: Reserved

Sources: [page_table_entry/src/arch/loongarch64.rs(L16 - L58)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/loongarch64.rs#L16-L58)

### Flag Conversion System

The LoongArch64 implementation provides bidirectional conversion between architecture-specific `PTEFlags` and generic `MappingFlags`:

```mermaid
flowchart TD
subgraph subGraph1["LoongArch64 to Generic"]
    PF["PTEFlags"]
    NR["!NR → READ"]
    W["W → WRITE"]
    NX["!NX → EXECUTE"]
    PLV["PLVL|PLVH → USER"]
    MAT["MAT bits → DEVICE/UNCACHED"]
    MF["MappingFlags"]
    READ["READ → !NR"]
    WRITE["WRITE → W"]
    EXECUTE["EXECUTE → !NX"]
    USER["USER → PLVH|PLVL"]
    DEVICE["DEVICE → MAT=00"]
end
subgraph subGraph0["Generic to LoongArch64"]
    PF["PTEFlags"]
    NR["!NR → READ"]
    W["W → WRITE"]
    NX["!NX → EXECUTE"]
    PLV["PLVL|PLVH → USER"]
    MAT["MAT bits → DEVICE/UNCACHED"]
    MF["MappingFlags"]
    READ["READ → !NR"]
    WRITE["WRITE → W"]
    EXECUTE["EXECUTE → !NX"]
    USER["USER → PLVH|PLVL"]
    DEVICE["DEVICE → MAT=00"]
    UNCACHED["UNCACHED → MAT=10"]
    CACHED["Default → MAT=01"]
end

MF --> CACHED
MF --> DEVICE
MF --> EXECUTE
MF --> READ
MF --> UNCACHED
MF --> USER
MF --> WRITE
PF --> MAT
PF --> NR
PF --> NX
PF --> PLV
PF --> W
```

Sources: [page_table_entry/src/arch/loongarch64.rs(L61 - L119)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/loongarch64.rs#L61-L119)

## Page Table Metadata

The `LA64MetaData` struct implements the `PagingMetaData` trait, providing architecture-specific constants and TLB management functionality.

### Page Walk Controller Configuration

LoongArch64 uses Page Walk Controllers (PWC) to configure hardware page table walking. The implementation defines specific CSR values for both lower and upper address space translation:

```mermaid
flowchart TD
subgraph subGraph2["PWCH Fields"]
    DIR3BASE["Dir3Base = 39Directory 3 Base"]
    DIR3WIDTH["Dir3Width = 9Directory 3 Width"]
    HPTW["HPTW_En = 0Hardware Page Table Walk"]
end
subgraph subGraph1["PWCL Fields"]
    PTBASE["PTBase = 12Page Table Base"]
    PTWIDTH["PTWidth = 9Page Table Width"]
    DIR1BASE["Dir1Base = 21Directory 1 Base"]
    DIR1WIDTH["Dir1Width = 9Directory 1 Width"]
    DIR2BASE["Dir2Base = 30Directory 2 Base"]
    DIR2WIDTH["Dir2Width = 9Directory 2 Width"]
end
subgraph subGraph0["PWC Configuration"]
    PWCL_REG["PWCL CSR (0x1C)Lower Half Address Space"]
    PWCH_REG["PWCH CSR (0x1D)Higher Half Address Space"]
end

PWCH_REG --> DIR3BASE
PWCH_REG --> DIR3WIDTH
PWCH_REG --> HPTW
PWCL_REG --> DIR1BASE
PWCL_REG --> DIR1WIDTH
PWCL_REG --> DIR2BASE
PWCL_REG --> DIR2WIDTH
PWCL_REG --> PTBASE
PWCL_REG --> PTWIDTH
```

Sources: [page_table_multiarch/src/arch/loongarch64.rs(L11 - L41)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/loongarch64.rs#L11-L41)

### TLB Management

The LoongArch64 implementation provides sophisticated TLB (Translation Lookaside Buffer) management using the `invtlb` instruction with data barrier synchronization:

```mermaid
flowchart TD
subgraph Synchronization["Synchronization"]
    DBAR["dbar 0Data BarrierMemory ordering"]
end
subgraph subGraph0["TLB Flush Operations"]
    FLUSH_ALL["flush_tlb(None)Clear all entriesinvtlb 0x00"]
    FLUSH_ADDR["flush_tlb(Some(vaddr))Clear specific entryinvtlb 0x05"]
end

DBAR --> FLUSH_ADDR
DBAR --> FLUSH_ALL
```

The implementation uses:

* `dbar 0`: Data barrier ensuring memory ordering
* `invtlb 0x00`: Clear all page table entries
* `invtlb 0x05`: Clear entries matching specific virtual address with G=0

Sources: [page_table_multiarch/src/arch/loongarch64.rs(L49 - L76)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/loongarch64.rs#L49-L76)

## Integration with Generic System

The LoongArch64 implementation seamlessly integrates with the generic trait system, providing the type alias `LA64PageTable` and implementing all required traits.

### Trait Implementation Structure

```mermaid
flowchart TD
subgraph subGraph2["Constants & Methods"]
    LEVELS["LEVELS = 4"]
    PA_BITS["PA_MAX_BITS = 48"]
    VA_BITS["VA_MAX_BITS = 48"]
    FLUSH["flush_tlb()"]
    NEW_PAGE["new_page()"]
    NEW_TABLE["new_table()"]
    FLAGS["flags()"]
    PADDR["paddr()"]
end
subgraph subGraph1["LoongArch64 Implementation"]
    LA64MD["LA64MetaData"]
    LA64PTE["LA64PTE"]
    LA64PT["LA64PageTable<I>"]
end
subgraph subGraph0["Generic Traits"]
    PMD["PagingMetaData trait"]
    GPTE["GenericPTE trait"]
    PT64["PageTable64<M,PTE,H>"]
end

GPTE --> FLAGS
GPTE --> NEW_PAGE
GPTE --> NEW_TABLE
GPTE --> PADDR
LA64MD --> PMD
LA64PT --> PT64
LA64PTE --> GPTE
PMD --> FLUSH
PMD --> LEVELS
PMD --> PA_BITS
PMD --> VA_BITS
```

Sources: [page_table_multiarch/src/arch/loongarch64.rs(L43 - L76)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/loongarch64.rs#L43-L76) [page_table_entry/src/arch/loongarch64.rs(L135 - L177)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/loongarch64.rs#L135-L177) [page_table_multiarch/src/arch/loongarch64.rs(L85)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/loongarch64.rs#L85-L85)

### Type Aliases and Exports

The LoongArch64 support is exposed through a clean type alias that follows the library's naming conventions:

```
pub type LA64PageTable<I> = PageTable64<LA64MetaData, LA64PTE, I>;
```

This allows users to instantiate LoongArch64 page tables with their preferred `PagingHandler` implementation while maintaining full compatibility with the generic `PageTable64` interface.

Sources: [page_table_multiarch/src/arch/loongarch64.rs(L78 - L85)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/loongarch64.rs#L78-L85)