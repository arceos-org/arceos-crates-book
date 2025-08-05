# AArch64 Support

> **Relevant source files**
> * [page_table_entry/src/arch/aarch64.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/aarch64.rs)
> * [page_table_multiarch/src/arch/aarch64.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/aarch64.rs)

This document covers the AArch64 (ARM64) architecture implementation in the page_table_multiarch library, including VMSAv8-64 translation table format support, memory attributes, and AArch64-specific page table operations. For information about other supported architectures, see [Architecture Support](/arceos-org/page_table_multiarch/4-architecture-support).

## Overview

The AArch64 implementation provides support for the VMSAv8-64 translation table format used by ARM64 processors. The implementation consists of two main components: the low-level page table entry definitions in the `page_table_entry` crate and the high-level page table metadata and operations in the `page_table_multiarch` crate.

**AArch64 Implementation Architecture**

```mermaid
flowchart TD
subgraph subGraph3["Hardware Interaction"]
    TLBI["TLBI instructions"]
    MAIR_EL1_reg["MAIR_EL1 register"]
    VMSAv8["VMSAv8-64 format"]
end
subgraph subGraph2["Core Traits"]
    PagingMetaData_trait["PagingMetaData"]
    GenericPTE_trait["GenericPTE"]
    PageTable64_trait["PageTable64"]
end
subgraph page_table_entry/src/arch/aarch64.rs["page_table_entry/src/arch/aarch64.rs"]
    A64PTE["A64PTE"]
    DescriptorAttr["DescriptorAttr"]
    MemAttr["MemAttr"]
    MAIR_VALUE["MAIR_VALUE"]
end
subgraph page_table_multiarch/src/arch/aarch64.rs["page_table_multiarch/src/arch/aarch64.rs"]
    A64PageTable["A64PageTable<H>"]
    A64PagingMetaData["A64PagingMetaData"]
    FlushTLB["flush_tlb()"]
end

A64PTE --> DescriptorAttr
A64PTE --> GenericPTE_trait
A64PTE --> VMSAv8
A64PageTable --> PageTable64_trait
A64PagingMetaData --> FlushTLB
A64PagingMetaData --> PagingMetaData_trait
DescriptorAttr --> MemAttr
DescriptorAttr --> VMSAv8
FlushTLB --> TLBI
MAIR_VALUE --> MAIR_EL1_reg
MemAttr --> MAIR_VALUE
```

Sources: [page_table_multiarch/src/arch/aarch64.rs(L1 - L38)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/aarch64.rs#L1-L38) [page_table_entry/src/arch/aarch64.rs(L1 - L265)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/aarch64.rs#L1-L265)

## Page Table Entry Implementation

The `A64PTE` struct implements the `GenericPTE` trait and represents VMSAv8-64 translation table descriptors. It uses the `DescriptorAttr` bitflags to manage AArch64-specific memory attributes and access permissions.

**AArch64 Page Table Entry Structure**

```mermaid
flowchart TD
subgraph subGraph3["Memory Attributes"]
    MemAttr_enum["MemAttr enum"]
    Device["Device (0)"]
    Normal["Normal (1)"]
    NormalNonCacheable["NormalNonCacheable (2)"]
end
subgraph subGraph2["GenericPTE Methods"]
    new_page["new_page()"]
    new_table["new_table()"]
    paddr["paddr()"]
    flags["flags()"]
    is_huge["is_huge()"]
    is_present["is_present()"]
end
subgraph subGraph1["DescriptorAttr Bitflags"]
    VALID["VALID (bit 0)"]
    NON_BLOCK["NON_BLOCK (bit 1)"]
    ATTR_INDX["ATTR_INDX (bits 2-4)"]
    AP_EL0["AP_EL0 (bit 6)"]
    AP_RO["AP_RO (bit 7)"]
    AF["AF (bit 10)"]
    PXN["PXN (bit 53)"]
    UXN["UXN (bit 54)"]
end
subgraph subGraph0["A64PTE Structure"]
    A64PTE_struct["A64PTE(u64)"]
    PHYS_ADDR_MASK["PHYS_ADDR_MASK0x0000_ffff_ffff_f000bits 12..48"]
end

A64PTE_struct --> flags
A64PTE_struct --> is_huge
A64PTE_struct --> is_present
A64PTE_struct --> new_page
A64PTE_struct --> new_table
A64PTE_struct --> paddr
AF --> new_page
AP_EL0 --> flags
AP_RO --> flags
ATTR_INDX --> MemAttr_enum
MemAttr_enum --> Device
MemAttr_enum --> Normal
MemAttr_enum --> NormalNonCacheable
NON_BLOCK --> is_huge
PHYS_ADDR_MASK --> paddr
PXN --> flags
UXN --> flags
VALID --> is_present
```

Sources: [page_table_entry/src/arch/aarch64.rs(L191 - L253)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/aarch64.rs#L191-L253) [page_table_entry/src/arch/aarch64.rs(L9 - L58)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/aarch64.rs#L9-L58)

### Key Methods and Bit Manipulation

The `A64PTE` implementation provides several key methods for page table entry manipulation:

|Method|Purpose|Physical Address Mask|
| --- | --- | --- |
|new_page()|Creates page descriptor with mapping flags|0x0000_ffff_ffff_f000|
|new_table()|Creates table descriptor for next level|0x0000_ffff_ffff_f000|
|paddr()|Extracts physical address from bits 12-47|0x0000_ffff_ffff_f000|
|is_huge()|Checks if NON_BLOCK bit is clear|N/A|
|is_present()|Checks VALID bit|N/A|

Sources: [page_table_entry/src/arch/aarch64.rs(L200 - L253)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/aarch64.rs#L200-L253)

## Memory Attributes and MAIR Configuration

The AArch64 implementation uses the Memory Attribute Indirection Register (MAIR) to define memory types. The `MemAttr` enum defines three memory attribute types that correspond to MAIR indices.

**Memory Attribute Configuration**

```mermaid
flowchart TD
subgraph subGraph2["DescriptorAttr Conversion"]
    from_mem_attr["from_mem_attr()"]
    INNER_SHAREABLE["INNER + SHAREABLEfor Normal memory"]
end
subgraph subGraph1["MAIR_EL1 Register Values"]
    MAIR_VALUE["MAIR_VALUE: 0x44_ff_04"]
    attr0["attr0: Device-nGnREnonGathering_nonReordering_EarlyWriteAck"]
    attr1["attr1: WriteBack_NonTransient_ReadWriteAllocInner + Outer"]
    attr2["attr2: NonCacheableInner + Outer"]
end
subgraph subGraph0["MemAttr Types"]
    Device_attr["Device (index 0)Device-nGnRE"]
    Normal_attr["Normal (index 1)WriteBack cacheable"]
    NormalNC_attr["NormalNonCacheable (index 2)Non-cacheable"]
end

Device_attr --> attr0
Device_attr --> from_mem_attr
NormalNC_attr --> attr2
NormalNC_attr --> from_mem_attr
Normal_attr --> attr1
Normal_attr --> from_mem_attr
attr0 --> MAIR_VALUE
attr1 --> MAIR_VALUE
attr2 --> MAIR_VALUE
from_mem_attr --> INNER_SHAREABLE
```

Sources: [page_table_entry/src/arch/aarch64.rs(L62 - L112)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/aarch64.rs#L62-L112) [page_table_entry/src/arch/aarch64.rs(L73 - L97)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/aarch64.rs#L73-L97)

### Flag Conversion

The implementation provides bidirectional conversion between generic `MappingFlags` and AArch64-specific `DescriptorAttr`:

**Flag Mapping Table**

|MappingFlags|DescriptorAttr|Description|
| --- | --- | --- |
|READ|VALID|Entry is valid and readable|
|WRITE|!AP_RO|Write access permitted (when AP_RO is clear)|
|EXECUTE|!PXNor!UXN|Execute permissions based on privilege level|
|USER|AP_EL0|Accessible from EL0 (user space)|
|DEVICE|MemAttr::Device|Device memory with ATTR_INDX = 0|
|UNCACHED|MemAttr::NormalNonCacheable|Non-cacheable normal memory|

Sources: [page_table_entry/src/arch/aarch64.rs(L114 - L189)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/aarch64.rs#L114-L189)

## Page Table Metadata

The `A64PagingMetaData` struct implements the `PagingMetaData` trait and defines AArch64-specific constants and operations for the VMSAv8-64 translation scheme.

### Configuration Constants

|Constant|Value|Description|
| --- | --- | --- |
|LEVELS|4|Number of page table levels|
|PA_MAX_BITS|48|Maximum physical address bits|
|VA_MAX_BITS|48|Maximum virtual address bits|

Sources: [page_table_multiarch/src/arch/aarch64.rs(L11 - L15)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/aarch64.rs#L11-L15)

### Virtual Address Validation

The `vaddr_is_valid()` method implements AArch64's canonical address validation:

```javascript
fn vaddr_is_valid(vaddr: usize) -> bool {
    let top_bits = vaddr >> Self::VA_MAX_BITS;
    top_bits == 0 || top_bits == 0xffff
}
```

This ensures that virtual addresses are either in the lower canonical range (top 16 bits all zeros) or upper canonical range (top 16 bits all ones).

Sources: [page_table_multiarch/src/arch/aarch64.rs(L17 - L20)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/aarch64.rs#L17-L20)

## TLB Management

The AArch64 implementation provides TLB (Translation Lookaside Buffer) invalidation through assembly instructions in the `flush_tlb()` method.

**TLB Invalidation Operations**

```mermaid
flowchart TD
subgraph subGraph0["flush_tlb() Implementation"]
    flush_tlb_method["flush_tlb(vaddr: Option<VirtAddr>)"]
    vaddr_check["vaddr.is_some()?"]
    specific_tlbi["TLBI VAAE1ISTLB Invalidate by VAAll ASID, EL1, Inner Shareable"]
    global_tlbi["TLBI VMALLE1TLB Invalidate by VMIDAll at stage 1, EL1"]
    dsb_sy["DSB SYData Synchronization Barrier"]
    isb["ISBInstruction Synchronization Barrier"]
end

dsb_sy --> isb
flush_tlb_method --> vaddr_check
global_tlbi --> dsb_sy
specific_tlbi --> dsb_sy
vaddr_check --> global_tlbi
vaddr_check --> specific_tlbi
```

Sources: [page_table_multiarch/src/arch/aarch64.rs(L22 - L34)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/aarch64.rs#L22-L34)

### TLB Instruction Details

|Operation|Instruction|Scope|Description|
| --- | --- | --- | --- |
|Specific invalidation|tlbi vaae1is, {vaddr}|Single VA|Invalidates TLB entries for specific virtual address across all ASIDs|
|Global invalidation|tlbi vmalle1|All VAs|Invalidates all stage 1 TLB entries for current VMID|
|Memory barrier|dsb sy; isb|System-wide|Ensures TLB operations complete before subsequent instructions|

Sources: [page_table_multiarch/src/arch/aarch64.rs(L24 - L32)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/aarch64.rs#L24-L32)

## EL2 Support

The implementation includes conditional compilation support for ARM Exception Level 2 (EL2) through the `arm-el2` feature flag. This affects execute permission handling in the flag conversion logic.

### Permission Handling Differences

|Configuration|User Execute|Kernel Execute|Implementation|
| --- | --- | --- | --- |
|Withoutarm-el2|UsesUXNwhenAP_EL0set|UsesPXNfor kernel-only|Separate handling for user/kernel|
|Witharm-el2|UsesUXNonly|UsesUXNonly|Simplified execute control|

Sources: [page_table_entry/src/arch/aarch64.rs(L123 - L186)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/aarch64.rs#L123-L186)

## Type Alias and Integration

The `A64PageTable<H>` type alias provides the complete AArch64 page table implementation by combining the metadata, page table entry type, and handler:

```
pub type A64PageTable<H> = PageTable64<A64PagingMetaData, A64PTE, H>;
```

This creates a fully configured page table type that applications can use with their chosen `PagingHandler` implementation.

Sources: [page_table_multiarch/src/arch/aarch64.rs(L37)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/aarch64.rs#L37-L37)