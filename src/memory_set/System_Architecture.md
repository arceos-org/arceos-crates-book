# System Architecture

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md)
> * [src/set.rs](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs)

This document details the architectural design of the memory_set crate, focusing on how the core components interact, the generic type system design, and the data flow through memory management operations. For an introduction to the fundamental concepts of `MemorySet`, `MemoryArea`, and `MappingBackend`, see [Core Concepts](/arceos-org/memory_set/1.1-core-concepts). For detailed implementation specifics of individual components, see [Implementation Details](/arceos-org/memory_set/2-implementation-details).

## Component Architecture and Interactions

The memory_set crate follows a layered architecture with three primary abstraction levels: the collection layer (`MemorySet`), the area management layer (`MemoryArea`), and the backend interface layer (`MappingBackend`).

**Component Interaction Overview**

```mermaid
flowchart TD
subgraph subGraph3["External Dependencies"]
    PageTable["Page Table (P)"]
    MemoryAddr["memory_addr crate"]
end
subgraph subGraph2["Backend Interface Layer"]
    MappingBackend["MappingBackend<F,P> trait"]
    ConcreteBackend["Concrete Backend Implementation"]
end
subgraph subGraph1["Area Management Layer"]
    MemoryArea1["MemoryArea<F,P,B>"]
    MemoryArea2["MemoryArea<F,P,B>"]
    VirtAddrRange["VirtAddrRange"]
end
subgraph subGraph0["Collection Layer"]
    MemorySet["MemorySet<F,P,B>"]
    BTreeStorage["BTreeMap<VirtAddr, MemoryArea>"]
end

BTreeStorage --> MemoryArea1
BTreeStorage --> MemoryArea2
ConcreteBackend --> MappingBackend
MappingBackend --> PageTable
MemoryArea1 --> MappingBackend
MemoryArea1 --> VirtAddrRange
MemoryArea2 --> MappingBackend
MemorySet --> BTreeStorage
MemorySet --> PageTable
VirtAddrRange --> MemoryAddr
```

The `MemorySet` acts as the orchestrator, managing collections of `MemoryArea` objects through a `BTreeMap` indexed by virtual addresses. Each `MemoryArea` delegates actual page table manipulation to its associated `MappingBackend` implementation.

**Sources:** [src/set.rs(L9 - L11)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L9-L11) [README.md(L18 - L41)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L18-L41)

## Generic Type System Design

The crate achieves flexibility through a carefully designed generic type system that allows different flag types, page table implementations, and backend strategies while maintaining type safety.

**Generic Type Parameter Relationships**

```mermaid
flowchart TD
subgraph subGraph2["Concrete Mock Example"]
    MockFlags["MockFlags = u8"]
    MockPageTable["MockPageTable = [u8; MAX_ADDR]"]
    MockBackend["MockBackend"]
    ConcreteMemorySet["MemorySet<u8, [u8; MAX_ADDR], MockBackend>"]
end
subgraph subGraph1["Core Generic Types"]
    MemorySet["MemorySet<F,P,B>"]
    MemoryArea["MemoryArea<F,P,B>"]
    MappingBackendTrait["MappingBackend<F,P>"]
end
subgraph subGraph0["Generic Parameters"]
    F["F: Copy (Flags Type)"]
    P["P (Page Table Type)"]
    B["B: MappingBackend<F,P> (Backend Type)"]
end

B --> MappingBackendTrait
B --> MemoryArea
B --> MemorySet
F --> MappingBackendTrait
F --> MemoryArea
F --> MemorySet
MockBackend --> ConcreteMemorySet
MockBackend --> MappingBackendTrait
MockFlags --> ConcreteMemorySet
MockPageTable --> ConcreteMemorySet
P --> MappingBackendTrait
P --> MemoryArea
P --> MemorySet
```

The type parameter `F` represents memory flags and must implement `Copy`. The page table type `P` is completely generic, allowing integration with different page table implementations. The backend type `B` must implement `MappingBackend<F,P>`, creating a three-way constraint that ensures type compatibility across the entire system.

**Sources:** [src/set.rs(L9)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L9-L9) [README.md(L24 - L31)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L24-L31)

## Memory Management Data Flow

Memory management operations follow predictable patterns that involve coordination between all architectural layers. The most complex operations, such as unmapping that splits existing areas, demonstrate the sophisticated interaction patterns.

**Map Operation Data Flow**

```mermaid
sequenceDiagram
    participant Client as Client
    participant MemorySet as MemorySet
    participant BTreeMap as BTreeMap
    participant MemoryArea as MemoryArea
    participant MappingBackend as MappingBackend
    participant PageTable as PageTable

    Client ->> MemorySet: "map(area, page_table, unmap_overlap)"
    MemorySet ->> MemorySet: "overlaps() check"
    alt overlaps && unmap_overlap
        MemorySet ->> MemorySet: "unmap() existing regions"
        MemorySet ->> MappingBackend: "unmap() calls"
        MappingBackend ->> PageTable: "clear entries"
    else overlaps && !unmap_overlap
        MemorySet -->> Client: "MappingError::AlreadyExists"
    end
    MemorySet ->> MemoryArea: "map_area(page_table)"
    MemoryArea ->> MappingBackend: "map() call"
    MappingBackend ->> PageTable: "set entries"
    MappingBackend -->> MemoryArea: "success/failure"
    MemoryArea -->> MemorySet: "MappingResult"
    MemorySet ->> BTreeMap: "insert(area.start(), area)"
    MemorySet -->> Client: "MappingResult"
```

**Unmap Operation with Area Splitting**

```mermaid
sequenceDiagram
    participant Client as Client
    participant MemorySet as MemorySet
    participant BTreeMap as BTreeMap
    participant MemoryArea as MemoryArea
    participant MappingBackend as MappingBackend

    Client ->> MemorySet: "unmap(start, size, page_table)"
    MemorySet ->> BTreeMap: "find affected areas"
    loop for each affected area
    alt area fully contained
        MemorySet ->> BTreeMap: "remove area"
        MemorySet ->> MemoryArea: "unmap_area()"
    else area intersects left boundary
        MemorySet ->> MemoryArea: "shrink_right()"
    else area intersects right boundary
        MemorySet ->> MemoryArea: "shrink_left()"
    else unmap range in middle
        MemorySet ->> MemoryArea: "split(end_addr)"
        MemoryArea -->> MemorySet: "right_part created"
        MemorySet ->> MemoryArea: "shrink_right() on left part"
        MemorySet ->> BTreeMap: "insert(end, right_part)"
    end
    MemorySet ->> MappingBackend: "unmap() call"
    end
    MemorySet -->> Client: "MappingResult"
```

The unmap operation demonstrates the most complex data flow, involving area splitting, shrinking, and careful coordination with the page table backend to maintain consistency.

**Sources:** [src/set.rs(L93 - L114)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L93-L114) [src/set.rs(L122 - L169)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L122-L169)

## Storage Organization and Efficiency

The `MemorySet` uses a `BTreeMap<VirtAddr, MemoryArea>` as its core storage mechanism, providing efficient operations for the common memory management use cases.

**BTreeMap Storage Structure**

```mermaid
flowchart TD
subgraph subGraph3["Key Operations"]
    OverlapDetection["overlaps() - O(log n)"]
    FindOperation["find() - O(log n)"]
    FreeAreaSearch["find_free_area() - O(n)"]
end
subgraph subGraph2["Efficiency Operations"]
    SortedOrder["Sorted by start address"]
    LogarithmicLookup["O(log n) lookups"]
    RangeQueries["Efficient range queries"]
end
subgraph subGraph1["BTreeMap Key-Value Organization"]
    Entry1["va!(0x1000) → MemoryArea[0x1000..0x2000]"]
    Entry2["va!(0x3000) → MemoryArea[0x3000..0x4000]"]
    Entry3["va!(0x5000) → MemoryArea[0x5000..0x6000]"]
end
subgraph subGraph0["MemorySet Internal Storage"]
    MemorySet["MemorySet"]
    Areas["areas: BTreeMap<VirtAddr, MemoryArea>"]
end

Areas --> Entry1
Areas --> Entry2
Areas --> Entry3
Entry1 --> SortedOrder
Entry2 --> LogarithmicLookup
Entry3 --> RangeQueries
LogarithmicLookup --> FindOperation
MemorySet --> Areas
RangeQueries --> FreeAreaSearch
SortedOrder --> OverlapDetection
```

The BTreeMap provides several efficiency advantages:

|Operation|Complexity|Implementation|
| --- | --- | --- |
|overlaps()|O(log n)|Range queries before/after target range|
|find()|O(log n)|Range query up to target address|
|find_free_area()|O(n)|Linear scan between existing areas|
|map()|O(log n)|Insert operation after overlap check|
|unmap()|O(log n + k)|Where k is the number of affected areas|

**Key Algorithm: Overlap Detection**

The overlap detection algorithm uses BTreeMap's range query capabilities to efficiently check for conflicts:

```mermaid
flowchart TD
subgraph subGraph0["Overlap Detection Strategy"]
    RangeBefore["range(..range.start).last()"]
    RangeAfter["range(range.start..).next()"]
    CheckBefore["Check if before.va_range().overlaps(range)"]
    CheckAfter["Check if after.va_range().overlaps(range)"]
end
Result["Boolean result"]

CheckAfter --> Result
CheckBefore --> Result
RangeAfter --> CheckAfter
RangeBefore --> CheckBefore
```

This approach requires at most two BTreeMap lookups regardless of the total number of areas, making it highly efficient even for large memory sets.

**Sources:** [src/set.rs(L10)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L10-L10) [src/set.rs(L37 - L49)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L37-L49) [src/set.rs(L52 - L55)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L52-L55) [src/set.rs(L64 - L83)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L64-L83)