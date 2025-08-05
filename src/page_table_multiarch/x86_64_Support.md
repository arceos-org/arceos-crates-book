# x86_64 Support

> **Relevant source files**
> * [page_table_entry/src/arch/x86_64.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/x86_64.rs)
> * [page_table_multiarch/src/arch/x86_64.rs](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/x86_64.rs)

This document covers the implementation of x86_64 architecture support in the page_table_multiarch library. It details how x86_64-specific page table entries, paging metadata, and memory management features are integrated with the generic multi-architecture abstraction layer.

For information about other supported architectures, see [AArch64 Support](/arceos-org/page_table_multiarch/4.2-aarch64-support), [RISC-V Support](/arceos-org/page_table_multiarch/4.3-risc-v-support), and [LoongArch64 Support](/arceos-org/page_table_multiarch/4.4-loongarch64-support). For details about the overall architecture abstraction system, see [Generic Traits System](/arceos-org/page_table_multiarch/3.2-generic-traits-system).

## x86_64 Architecture Overview

The x86_64 implementation supports Intel and AMD 64-bit processors using 4-level page translation with the following characteristics:

|Property|Value|Description|
| --- | --- | --- |
|Paging Levels|4|PML4, PDPT, PD, PT|
|Physical Address Space|52 bits|Up to 4 PB of physical memory|
|Virtual Address Space|48 bits|256 TB of virtual memory|
|Page Sizes|4KB, 2MB, 1GB|Standard and huge page support|

### x86_64 Paging Hierarchy

```mermaid
flowchart TD
subgraph subGraph0["Virtual Address Breakdown"]
    VA["48-bit Virtual Address"]
    PML4_IDX["Bits 47-39PML4 Index"]
    PDPT_IDX["Bits 38-30PDPT Index"]
    PD_IDX["Bits 29-21PD Index"]
    PT_IDX["Bits 20-12PT Index"]
    OFFSET["Bits 11-0Page Offset"]
end
CR3["CR3 RegisterPML4 Base Address"]
PML4["PML4 (Level 4)Page Map Level 4"]
PDPT["PDPT (Level 3)Page Directory Pointer Table"]
PD["PD (Level 2)Page Directory"]
PT["PT (Level 1)Page Table"]
Page["4KB Page"]
HugePage2M["2MB Huge Page"]
HugePage1G["1GB Huge Page"]

CR3 --> PML4
PD --> HugePage2M
PD --> PT
PDPT --> HugePage1G
PDPT --> PD
PML4 --> PDPT
PT --> Page
VA --> OFFSET
VA --> PDPT_IDX
VA --> PD_IDX
VA --> PML4_IDX
VA --> PT_IDX
```

Sources: [page_table_multiarch/src/arch/x86_64.rs(L10 - L12)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/x86_64.rs#L10-L12)

## Page Table Entry Implementation

The `X64PTE` struct implements x86_64-specific page table entries with a 64-bit representation that follows Intel's page table entry format.

### X64PTE Structure and Bit Layout

```mermaid
flowchart TD
subgraph subGraph2["GenericPTE Implementation"]
    NEW_PAGE["new_page()"]
    NEW_TABLE["new_table()"]
    PADDR["paddr()"]
    FLAGS_METHOD["flags()"]
    SET_METHODS["set_paddr(), set_flags()"]
    STATE_METHODS["is_present(), is_huge(), is_unused()"]
end
subgraph subGraph0["X64PTE Bit Layout (64 bits)"]
    RESERVED["Bits 63-52Reserved/Available"]
    PHYS_ADDR["Bits 51-12Physical Address(PHYS_ADDR_MASK)"]
    FLAGS["Bits 11-0Page Table Flags"]
end
subgraph subGraph1["Key Flag Bits"]
    PRESENT["Bit 0: Present"]
    WRITABLE["Bit 1: Writable"]
    USER["Bit 2: User Accessible"]
    WRITE_THROUGH["Bit 3: Write Through"]
    NO_CACHE["Bit 4: No Cache"]
    ACCESSED["Bit 5: Accessed"]
    DIRTY["Bit 6: Dirty"]
    HUGE_PAGE["Bit 7: Page Size (Huge)"]
    GLOBAL["Bit 8: Global"]
    NO_EXECUTE["Bit 63: No Execute (NX)"]
end

FLAGS --> FLAGS_METHOD
FLAGS_METHOD --> STATE_METHODS
NEW_PAGE --> SET_METHODS
PHYS_ADDR --> NEW_PAGE
```

The `X64PTE` struct uses a transparent representation over a `u64` value and implements the `GenericPTE` trait to provide architecture-neutral page table entry operations.

Sources: [page_table_entry/src/arch/x86_64.rs(L54 - L66)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/x86_64.rs#L54-L66) [page_table_entry/src/arch/x86_64.rs(L68 - L112)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/x86_64.rs#L68-L112)

### GenericPTE Trait Implementation

The `X64PTE` implements all required methods of the `GenericPTE` trait:

|Method|Purpose|Implementation Detail|
| --- | --- | --- |
|new_page()|Create page mapping entry|Sets physical address and convertsMappingFlagstoPTF|
|new_table()|Create page table entry|SetsPRESENT,WRITABLE,USER_ACCESSIBLEflags|
|paddr()|Extract physical address|Masks bits 12-51 usingPHYS_ADDR_MASK|
|flags()|Get mapping flags|ConvertsPTFflags to genericMappingFlags|
|is_present()|Check if entry is valid|TestsPTF::PRESENTbit|
|is_huge()|Check if huge page|TestsPTF::HUGE_PAGEbit|

Sources: [page_table_entry/src/arch/x86_64.rs(L69 - L95)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/x86_64.rs#L69-L95) [page_table_entry/src/arch/x86_64.rs(L97 - L112)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/x86_64.rs#L97-L112)

## Paging Metadata Implementation

The `X64PagingMetaData` struct defines x86_64-specific constants and operations required by the `PagingMetaData` trait.

### Architecture Constants and TLB Management

```mermaid
flowchart TD
subgraph subGraph2["PageTable64 Integration"]
    PT64["PageTable64"]
    TYPE_ALIAS["X64PageTable"]
end
subgraph subGraph0["X64PagingMetaData Constants"]
    LEVELS["LEVELS = 44-level page tables"]
    PA_BITS["PA_MAX_BITS = 5252-bit physical addresses"]
    VA_BITS["VA_MAX_BITS = 4848-bit virtual addresses"]
    VIRT_ADDR["VirtAddr = memory_addr::VirtAddrVirtual address type"]
end
subgraph subGraph1["TLB Management"]
    FLUSH_TLB["flush_tlb()"]
    SINGLE_FLUSH["x86::tlb::flush(vaddr)Single page invalidation"]
    FULL_FLUSH["x86::tlb::flush_all()Complete TLB flush"]
end

FLUSH_TLB --> FULL_FLUSH
FLUSH_TLB --> SINGLE_FLUSH
LEVELS --> PT64
PA_BITS --> PT64
PT64 --> TYPE_ALIAS
VA_BITS --> PT64
VIRT_ADDR --> PT64
```

The TLB flushing implementation uses the `x86` crate to perform architecture-specific translation lookaside buffer invalidation operations safely.

Sources: [page_table_multiarch/src/arch/x86_64.rs(L7 - L25)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/x86_64.rs#L7-L25) [page_table_multiarch/src/arch/x86_64.rs(L28)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/x86_64.rs#L28-L28)

## Flag Conversion System

The x86_64 implementation provides bidirectional conversion between generic `MappingFlags` and x86-specific `PageTableFlags` (PTF).

### MappingFlags to PTF Conversion

|Generic Flag|x86_64 PTF|Notes|
| --- | --- | --- |
|READ|PRESENT|Always set for readable pages|
|WRITE|WRITABLE|Write permission|
|EXECUTE|!NO_EXECUTE|Execute permission (inverted logic)|
|USER|USER_ACCESSIBLE|User-mode access|
|UNCACHED|NO_CACHE|Disable caching|
|DEVICE|NO_CACHE \| WRITE_THROUGH|Device memory attributes|

### PTF to MappingFlags Conversion

The reverse conversion extracts generic flags from x86-specific page table flags, handling the inverted execute permission logic and presence requirements.

```mermaid
flowchart TD
subgraph subGraph1["Special Handling"]
    EMPTY_CHECK["Empty flags → Empty PTF"]
    PRESENT_REQ["Non-empty flags → PRESENT bit set"]
    EXECUTE_INVERT["EXECUTE → !NO_EXECUTE"]
    DEVICE_COMBO["DEVICE → NO_CACHE | WRITE_THROUGH"]
end
subgraph subGraph0["Flag Conversion Flow"]
    GENERIC["MappingFlags(Architecture-neutral)"]
    X86_FLAGS["PageTableFlags (PTF)(x86-specific)"]
end

GENERIC --> DEVICE_COMBO
GENERIC --> EMPTY_CHECK
GENERIC --> EXECUTE_INVERT
GENERIC --> PRESENT_REQ
GENERIC --> X86_FLAGS
X86_FLAGS --> GENERIC
```

Sources: [page_table_entry/src/arch/x86_64.rs(L10 - L30)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/x86_64.rs#L10-L30) [page_table_entry/src/arch/x86_64.rs(L32 - L52)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_entry/src/arch/x86_64.rs#L32-L52)

## Integration with Generic System

The x86_64 support integrates with the generic page table system through the `X64PageTable` type alias, which specializes the generic `PageTable64` struct with x86_64-specific types.

### Type Composition

```mermaid
flowchart TD
subgraph subGraph2["Concrete Type"]
    X64_PAGE_TABLE["X64PageTable =PageTable64"]
end
subgraph subGraph1["x86_64 Implementations"]
    X64_METADATA["X64PagingMetaData"]
    X64_PTE["X64PTE"]
    USER_HANDLER["H: PagingHandler(User-provided)"]
end
subgraph subGraph0["Generic Framework"]
    PT64_GENERIC["PageTable64"]
    METADATA_TRAIT["PagingMetaData trait"]
    PTE_TRAIT["GenericPTE trait"]
    HANDLER_TRAIT["PagingHandler trait"]
end

HANDLER_TRAIT --> USER_HANDLER
METADATA_TRAIT --> X64_METADATA
PT64_GENERIC --> X64_PAGE_TABLE
PTE_TRAIT --> X64_PTE
USER_HANDLER --> X64_PAGE_TABLE
X64_METADATA --> X64_PAGE_TABLE
X64_PTE --> X64_PAGE_TABLE
```

The `X64PageTable<H>` provides a complete x86_64 page table implementation that maintains all the functionality of the generic `PageTable64` while using x86_64-specific page table entries and metadata.

Sources: [page_table_multiarch/src/arch/x86_64.rs(L28)&emsp;](https://github.com/arceos-org/page_table_multiarch/blob/85fb75ef/page_table_multiarch/src/arch/x86_64.rs#L28-L28)