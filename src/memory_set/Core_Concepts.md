# Core Concepts

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md)

This document explains the fundamental building blocks of the memory_set crate: `MemorySet`, `MemoryArea`, and `MappingBackend`. These three types work together to provide a flexible abstraction for memory mapping operations similar to `mmap`, `munmap`, and `mprotect` system calls.

For detailed implementation specifics, see [Implementation Details](/arceos-org/memory_set/2-implementation-details). For practical usage examples, see [Basic Usage Patterns](/arceos-org/memory_set/3.1-basic-usage-patterns).

## The Three Core Types

The memory_set crate is built around three primary abstractions that work together to manage memory mappings:

```mermaid
flowchart TD
subgraph subGraph0["Generic Parameters"]
    F["F: Memory Flags"]
    PT["PT: Page Table Type"]
    B["B: Backend Implementation"]
end
MemorySet["MemorySet"]
MemoryArea["MemoryArea"]
MappingBackend["MappingBackend"]
BTreeMap["BTreeMap"]
VirtAddrRange["VirtAddrRange"]
PageTable["Page Table (PT)"]

B --> MemorySet
F --> MemorySet
MappingBackend --> PageTable
MemoryArea --> MappingBackend
MemoryArea --> VirtAddrRange
MemorySet --> BTreeMap
MemorySet --> MappingBackend
MemorySet --> MemoryArea
PT --> MemorySet
```

Sources: [README.md(L18 - L41)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L18-L41)

## MemorySet: Collection Management

`MemorySet<F, PT, B>` serves as the primary interface for managing collections of memory areas. It maintains a sorted collection of non-overlapping memory regions and provides high-level operations for mapping, unmapping, and protecting memory.

### Key Responsibilities

|Operation|Description|Overlap Handling|
| --- | --- | --- |
|map()|Add new memory area|Can split/remove existing areas|
|unmap()|Remove memory regions|Automatically splits affected areas|
|protect()|Change permissions|Updates flags for matching areas|
|iter()|Enumerate areas|Provides ordered traversal|

The `MemorySet` uses a `BTreeMap<VirtAddr, MemoryArea>` internally to maintain areas sorted by their starting virtual address, enabling efficient overlap detection and range queries.

```mermaid
flowchart TD
subgraph Operations["Operations"]
    VR1["VirtAddrRange: 0x1000-0x2000"]
    VR2["VirtAddrRange: 0x3000-0x4000"]
    VR3["VirtAddrRange: 0x5000-0x6000"]
end
subgraph subGraph1["Area Organization"]
    A1["va!(0x1000) -> MemoryArea"]
    A2["va!(0x3000) -> MemoryArea"]
    A3["va!(0x5000) -> MemoryArea"]
end
subgraph subGraph0["MemorySet Internal Structure"]
    MS["MemorySet"]
    BT["BTreeMap"]
end

A1 --> VR1
A2 --> VR2
A3 --> VR3
BT --> A1
BT --> A2
BT --> A3
MS --> BT
```

Sources: [README.md(L34 - L48)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L34-L48)

## MemoryArea: Individual Memory Regions

`MemoryArea<F, B>` represents a contiguous region of virtual memory with specific properties including address range, permissions flags, and an associated backend for page table operations.

### Core Properties

Each `MemoryArea` encapsulates:

* **Virtual Address Range**: Start address and size defining the memory region
* **Flags**: Memory permissions (read, write, execute, etc.)
* **Backend**: Implementation for actual page table manipulation

### Area Lifecycle Operations

```

```

Sources: [README.md(L37 - L38)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L37-L38)

## MappingBackend: Page Table Interface

The `MappingBackend<F, PT>` trait defines the interface between memory areas and the underlying page table implementation. This abstraction allows the memory_set crate to work with different page table formats and memory management systems.

### Required Operations

All backends must implement three core operations:

|Method|Parameters|Purpose|
| --- | --- | --- |
|map()|start: VirtAddr, size: usize, flags: F, pt: &mut PT|Establish new memory mappings|
|unmap()|start: VirtAddr, size: usize, pt: &mut PT|Remove existing mappings|
|protect()|start: VirtAddr, size: usize, new_flags: F, pt: &mut PT|Change mapping permissions|

### Example Implementation Pattern

The mock backend demonstrates the interface pattern:

```mermaid
flowchart TD
subgraph subGraph1["MappingBackend Trait"]
    map_method["map()"]
    unmap_method["unmap()"]
    protect_method["protect()"]
end
subgraph subGraph0["MockBackend Implementation"]
    MockBackend["MockBackend struct"]
    MockFlags["MockFlags = u8"]
    MockPageTable["MockPageTable = [u8; MAX_ADDR]"]
end

MockBackend --> map_method
MockBackend --> protect_method
MockBackend --> unmap_method
MockFlags --> MockPageTable
map_method --> MockPageTable
protect_method --> MockPageTable
unmap_method --> MockPageTable
```

Sources: [README.md(L51 - L87)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L51-L87)

## Generic Type System

The memory_set crate uses a sophisticated generic type system to provide flexibility while maintaining type safety:

### Type Parameters

* **F**: Memory flags type (must implement `Copy`)
* **PT**: Page table type (can be any structure)
* **B**: Backend implementation (must implement `MappingBackend<F, PT>`)

This design allows the crate to work with different:

* Flag representations (bitfields, enums, integers)
* Page table formats (arrays, trees, hardware tables)
* Backend strategies (direct manipulation, system calls, simulation)

```mermaid
flowchart TD
subgraph Usage["Usage"]
    MemorySet_Concrete["MemorySet"]
end
subgraph subGraph1["Concrete Example"]
    MockFlags_u8["MockFlags = u8"]
    MockPT_Array["MockPageTable = [u8; MAX_ADDR]"]
    MockBackend_Impl["MockBackend: MappingBackend"]
end
subgraph subGraph0["Generic Constraints"]
    F_Copy["F: Copy"]
    B_Backend["B: MappingBackend"]
    PT_Any["PT: Any type"]
end

B_Backend --> MockBackend_Impl
F_Copy --> MockFlags_u8
MockBackend_Impl --> MemorySet_Concrete
MockFlags_u8 --> MemorySet_Concrete
MockPT_Array --> MemorySet_Concrete
PT_Any --> MockPT_Array
```

Sources: [README.md(L24 - L31)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L24-L31)

## Coordinated Operation Flow

The three core types work together in a coordinated fashion to handle memory management operations:

```mermaid
sequenceDiagram
    participant Client as Client
    participant MemorySet as MemorySet
    participant MemoryArea as MemoryArea
    participant MappingBackend as MappingBackend
    participant PageTable as PageTable

    Note over Client,PageTable: Memory Mapping Operation
    Client ->> MemorySet: "map(area, page_table, unmap_overlap)"
    MemorySet ->> MemorySet: "check for overlapping areas"
    alt overlaps exist and unmap_overlap=true
        MemorySet ->> MemoryArea: "split/shrink existing areas"
        MemorySet ->> MappingBackend: "unmap() overlapping regions"
        MappingBackend ->> PageTable: "clear page table entries"
    end
    MemorySet ->> MemoryArea: "get backend reference"
    MemoryArea ->> MappingBackend: "map(start, size, flags, pt)"
    MappingBackend ->> PageTable: "set page table entries"
    MappingBackend -->> MemorySet: "return success/failure"
    MemorySet ->> MemorySet: "insert area into BTreeMap"
    MemorySet -->> Client: "return MappingResult"
    Note over Client,PageTable: Memory Unmapping Operation
    Client ->> MemorySet: "unmap(start, size, page_table)"
    MemorySet ->> MemorySet: "find affected areas"
    loop for each affected area
        MemorySet ->> MemoryArea: "determine overlap relationship"
    alt area fully contained
        MemorySet ->> MemorySet: "remove area from BTreeMap"
    else area partially overlapping
        MemorySet ->> MemoryArea: "split area at boundaries"
        MemorySet ->> MemorySet: "update BTreeMap with new areas"
    end
    MemorySet ->> MappingBackend: "unmap(start, size, pt)"
    MappingBackend ->> PageTable: "clear page table entries"
    end
    MemorySet -->> Client: "return MappingResult"
```

Sources: [README.md(L42 - L43)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L42-L43)