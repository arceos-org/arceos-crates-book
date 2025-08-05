# Implementation Details

> **Relevant source files**
> * [src/area.rs](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs)
> * [src/set.rs](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs)

This document provides detailed technical information about the internal implementation of the memory_set crate's core components. It covers the internal data structures, algorithms, and patterns used to implement memory mapping management functionality. For high-level concepts and architecture overview, see [System Architecture](/arceos-org/memory_set/1.2-system-architecture). For usage examples and public API documentation, see [Public API and Error Handling](/arceos-org/memory_set/2.3-public-api-and-error-handling).

## Core Data Structures and Types

The memory_set crate is built around three primary data structures that work together through a carefully designed generic type system.

### MemoryArea Internal Structure

The `MemoryArea` struct serves as the fundamental building block representing a contiguous virtual memory region with uniform properties.

```mermaid
flowchart TD
subgraph subGraph3["Key Methods"]
    MAPAREA["map_area()"]
    UNMAPAREA["unmap_area()"]
    PROTECTAREA["protect_area()"]
    SPLIT["split()"]
    SHRINKLEFT["shrink_left()"]
    SHRINKRIGHT["shrink_right()"]
end
subgraph subGraph2["Type Constraints"]
    FCOPY["F: Copy"]
    BMAPPING["B: MappingBackend"]
    PGENERIC["P: Page Table Type"]
end
subgraph subGraph1["VirtAddrRange Components"]
    START["start: VirtAddr"]
    END["end: VirtAddr"]
end
subgraph subGraph0["MemoryArea Fields"]
    MA["MemoryArea"]
    VAR["va_range: VirtAddrRange"]
    FLAGS["flags: F"]
    BACKEND["backend: B"]
    PHANTOM["_phantom: PhantomData<(F,P)>"]
end

BACKEND --> BMAPPING
FLAGS --> FCOPY
MA --> BACKEND
MA --> FLAGS
MA --> MAPAREA
MA --> PHANTOM
MA --> PROTECTAREA
MA --> SHRINKLEFT
MA --> SHRINKRIGHT
MA --> SPLIT
MA --> UNMAPAREA
MA --> VAR
PHANTOM --> PGENERIC
VAR --> END
VAR --> START
```

The `MemoryArea` struct maintains both metadata about the virtual address range and a reference to the backend that handles the actual page table manipulation. The `PhantomData` field ensures proper generic type relationships without runtime overhead.

**Sources:** [src/area.rs(L29 - L34)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs#L29-L34) [src/area.rs(L36 - L76)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs#L36-L76)

### MappingBackend Trait Implementation

The `MappingBackend` trait defines the interface for different memory mapping strategies, allowing the system to support various page table implementations and mapping policies.

```mermaid
flowchart TD
subgraph subGraph3["Implementation Requirements"]
    CLONE["Clone trait bound"]
    BOOLRESULT["Boolean success indicator"]
end
subgraph subGraph2["Generic Parameters"]
    F["F: Copy (Flags Type)"]
    P["P: Page Table Type"]
end
subgraph subGraph1["Backend Responsibilities"]
    MAPENTRY["Set page table entries"]
    CLEARENTRY["Clear page table entries"]
    UPDATEENTRY["Update entry permissions"]
end
subgraph subGraph0["MappingBackend Trait"]
    MB["MappingBackend"]
    MAP["map(start, size, flags, page_table) -> bool"]
    UNMAP["unmap(start, size, page_table) -> bool"]
    PROTECT["protect(start, size, new_flags, page_table) -> bool"]
end

F --> MB
MAP --> BOOLRESULT
MAP --> MAPENTRY
MB --> CLONE
MB --> MAP
MB --> PROTECT
MB --> UNMAP
P --> MB
PROTECT --> BOOLRESULT
PROTECT --> UPDATEENTRY
UNMAP --> BOOLRESULT
UNMAP --> CLEARENTRY
```

The trait's boolean return values allow backends to signal success or failure, which the higher-level operations convert into proper `MappingResult` types for error handling.

**Sources:** [src/area.rs(L15 - L22)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs#L15-L22) [src/area.rs(L90 - L110)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs#L90-L110)

### MemorySet Collection Structure

The `MemorySet` uses a `BTreeMap` to maintain an ordered collection of memory areas, enabling efficient range queries and overlap detection.

```mermaid
flowchart TD
subgraph subGraph3["Core Methods"]
    MAPMETHOD["map()"]
    UNMAPMETHOD["unmap()"]
    PROTECTMETHOD["protect()"]
    OVERLAPS["overlaps()"]
    FIND["find()"]
    FINDFREE["find_free_area()"]
end
subgraph subGraph2["Efficiency Operations"]
    RANGEQUERIES["O(log n) range queries"]
    OVERLAP["Efficient overlap detection"]
    SEARCH["Binary search for areas"]
end
subgraph subGraph1["BTreeMap Key-Value Organization"]
    KEY1["Key: area.start()"]
    VAL1["Value: MemoryArea"]
    ORDERING["Sorted by start address"]
end
subgraph subGraph0["MemorySet Structure"]
    MS["MemorySet"]
    BTREE["areas: BTreeMap>"]
end

BTREE --> KEY1
BTREE --> VAL1
KEY1 --> ORDERING
MS --> BTREE
MS --> FIND
MS --> FINDFREE
MS --> MAPMETHOD
MS --> OVERLAPS
MS --> PROTECTMETHOD
MS --> UNMAPMETHOD
ORDERING --> OVERLAP
ORDERING --> RANGEQUERIES
ORDERING --> SEARCH
```

The choice of `VirtAddr` as the key ensures that areas are naturally sorted by their start addresses, which is crucial for the overlap detection and range manipulation algorithms.

**Sources:** [src/set.rs(L9 - L11)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L9-L11) [src/set.rs(L36 - L49)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L36-L49)

## Memory Area Lifecycle Operations

Memory areas support sophisticated lifecycle operations that enable complex memory management patterns while maintaining consistency with the underlying page table.

### Area Splitting Algorithm

The `split()` method implements a critical operation for handling partial unmapping and protection changes. It divides a single memory area into two independent areas at a specified boundary.

```mermaid
sequenceDiagram
    participant Client as Client
    participant MemoryArea as "MemoryArea"
    participant MappingBackend as "MappingBackend"

    Note over Client,MappingBackend: split(pos: VirtAddr) Operation
    Client ->> MemoryArea: split(pos)
    MemoryArea ->> MemoryArea: Validate pos in range (start < pos < end)
    alt Valid position
        MemoryArea ->> MemoryArea: Create new_area from pos to end
        Note over MemoryArea: new_area = MemoryArea::new(pos, end-pos, flags, backend.clone())
        MemoryArea ->> MemoryArea: Shrink original to start..pos
        Note over MemoryArea: self.va_range.end = pos
        MemoryArea ->> Client: Return Some(new_area)
    else Invalid position
        MemoryArea ->> Client: Return None
    end
    Note over MemoryArea,MappingBackend: No page table operations during split
    Note over MemoryArea: Backend cloned, both areas share same mapping behavior
```

The splitting operation is purely metadata manipulation - it doesn't modify the page table entries. The actual page table changes happen when subsequent operations like `unmap()` or `protect()` are called on the split areas.

**Sources:** [src/area.rs(L148 - L163)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs#L148-L163)

### Area Shrinking Operations

Memory areas support shrinking from either end, which is essential for handling partial unmapping operations efficiently.

```mermaid
flowchart TD
subgraph subGraph2["Error Handling"]
    CHECKRESULT["Check backend success"]
    SUCCESS["Update metadata"]
    FAILURE["Return MappingError::BadState"]
end
subgraph subGraph1["shrink_right() Process"]
    SR["shrink_right(new_size)"]
    CALCUNMAP2["unmap_size = old_size - new_size"]
    UNMAPRIGHT["backend.unmap(start + new_size, unmap_size)"]
    UPDATEEND["va_range.end -= unmap_size"]
end
subgraph subGraph0["shrink_left() Process"]
    SL["shrink_left(new_size)"]
    CALCUNMAP["unmap_size = old_size - new_size"]
    UNMAPLEFT["backend.unmap(start, unmap_size)"]
    UPDATESTART["va_range.start += unmap_size"]
end

CALCUNMAP --> UNMAPLEFT
CALCUNMAP2 --> UNMAPRIGHT
CHECKRESULT --> FAILURE
CHECKRESULT --> SUCCESS
SL --> CALCUNMAP
SR --> CALCUNMAP2
UNMAPLEFT --> CHECKRESULT
UNMAPLEFT --> UPDATESTART
UNMAPRIGHT --> CHECKRESULT
UNMAPRIGHT --> UPDATEEND
```

Both shrinking operations immediately update the page table through the backend before modifying the area's metadata, ensuring consistency between the virtual memory layout and page table state.

**Sources:** [src/area.rs(L116 - L139)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs#L116-L139)

## Collection Management Algorithms

The `MemorySet` implements sophisticated algorithms for managing collections of memory areas, particularly for handling overlapping operations and maintaining area integrity.

### Overlap Detection Strategy

The overlap detection algorithm leverages the `BTreeMap`'s ordered structure to efficiently check for conflicts with minimal tree traversal.

```mermaid
flowchart TD
subgraph subGraph0["BTreeMap Range Queries"]
    BEFORECHECK["Find last area before range.start"]
    AFTERCHECK["Find first area >= range.start"]
    RANGEBEFORE["areas.range(..range.start).last()"]
    RANGEAFTER["areas.range(range.start..).next()"]
end
START["overlaps(range: VirtAddrRange)"]
BEFOREOVERLAP["Does before area overlap with range?"]
RETURNTRUE["return true"]
AFTEROVERLAP["Does after area overlap with range?"]
RETURNFALSE["return false"]

AFTERCHECK --> AFTEROVERLAP
AFTERCHECK --> RANGEAFTER
AFTEROVERLAP --> RETURNFALSE
AFTEROVERLAP --> RETURNTRUE
BEFORECHECK --> BEFOREOVERLAP
BEFORECHECK --> RANGEBEFORE
BEFOREOVERLAP --> AFTERCHECK
BEFOREOVERLAP --> RETURNTRUE
START --> BEFORECHECK
```

This algorithm achieves O(log n) complexity by examining at most two areas, regardless of the total number of areas in the set.

**Sources:** [src/set.rs(L36 - L49)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L36-L49)

### Complex Unmapping Algorithm

The `unmap()` operation handles the most complex scenario in memory management: removing an arbitrary address range that may affect multiple existing areas in different ways.

```mermaid
flowchart TD
subgraph subGraph3["Phase 3 Details"]
    RIGHTRANGE["areas.range_mut(start..).next()"]
    RIGHTINTERSECT["Check if after.start < end"]
    RIGHTCASE["Shrink left of area"]
end
subgraph subGraph2["Phase 2 Details"]
    LEFTRANGE["areas.range_mut(..start).last()"]
    LEFTINTERSECT["Check if before.end > start"]
    LEFTCASE1["Case 1: Unmap at end of area"]
    LEFTCASE2["Case 2: Unmap in middle (split required)"]
end
subgraph subGraph1["Phase 1 Details"]
    RETAIN["areas.retain() with condition"]
    CONTAINED["area.va_range().contained_in(range)"]
    REMOVE["Remove and unmap area"]
end
subgraph subGraph0["unmap() Three-Phase Algorithm"]
    PHASE1["Phase 1: Remove Fully Contained Areas"]
    PHASE2["Phase 2: Handle Left Boundary Intersection"]
    PHASE3["Phase 3: Handle Right Boundary Intersection"]
end

CONTAINED --> REMOVE
LEFTINTERSECT --> LEFTCASE1
LEFTINTERSECT --> LEFTCASE2
LEFTRANGE --> LEFTINTERSECT
PHASE1 --> RETAIN
PHASE2 --> LEFTRANGE
PHASE3 --> RIGHTRANGE
RETAIN --> CONTAINED
RIGHTINTERSECT --> RIGHTCASE
RIGHTRANGE --> RIGHTINTERSECT
```

This three-phase approach ensures that all possible area-range relationships are handled correctly, from simple removal to complex splitting scenarios.

**Sources:** [src/set.rs(L122 - L168)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L122-L168)

### Protection Changes with Area Management

The `protect()` operation demonstrates the most sophisticated area manipulation, potentially creating new areas while modifying existing ones.

```mermaid
sequenceDiagram
    participant MemorySet as "MemorySet"
    participant BTreeMap as "BTreeMap"
    participant MemoryArea as "MemoryArea"
    participant MappingBackend as "MappingBackend"
    participant to_insertVec as "to_insert: Vec"

    Note over MemorySet,to_insertVec: protect(start, size, update_flags, page_table)
    MemorySet ->> BTreeMap: Iterate through all areas
    loop For each area in range
        BTreeMap ->> MemorySet: area reference
        MemorySet ->> MemorySet: Call update_flags(area.flags())
    alt Flags should be updated
        Note over MemorySet,to_insertVec: Determine area-range relationship
    alt Area fully contained in range
        MemorySet ->> MemoryArea: protect_area(new_flags)
        MemoryArea ->> MappingBackend: protect(start, size, new_flags)
        MemorySet ->> MemoryArea: set_flags(new_flags)
    else Range fully contained in area (split into 3)
        MemorySet ->> MemoryArea: split(end) -> right_part
        MemorySet ->> MemoryArea: set_end(start) (becomes left_part)
        MemorySet ->> MemorySet: Create middle_part with new_flags
        MemorySet ->> to_insertVec: Queue right_part and middle_part for insertion
    else Partial overlaps
        MemorySet ->> MemoryArea: split() and protect as needed
        MemorySet ->> to_insertVec: Queue new parts for insertion
    end
    end
    end
    MemorySet ->> BTreeMap: areas.extend(to_insert)
```

The algorithm defers insertions to avoid iterator invalidation, collecting new areas in a vector and inserting them after the main iteration completes.

**Sources:** [src/set.rs(L189 - L247)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L189-L247)

## Generic Type System Implementation

The crate's generic type system enables flexible memory management while maintaining type safety and zero-cost abstractions.

### Type Parameter Relationships

```mermaid
flowchart TD
subgraph subGraph1["Type Flow Through System"]
    MAFLAG["MemoryArea.flags: F"]
    MBFLAG["MappingBackend.map(flags: F)"]
    MBPT["MappingBackend methods(page_table: &mut P)"]
    MABACKEND["MemoryArea.backend: B"]
    CLONE["Cloned for area.split()"]
end
subgraph subGraph0["Generic Type Constraints"]
    F["F: Copy + Debug"]
    P["P: Any"]
    B["B: MappingBackend + Clone"]
end
subgraph subGraph3["PhantomData Usage"]
    PHANTOM["PhantomData<(F,P)>"]
    TYPEREL["Maintains F,P relationship in MemoryArea"]
    ZEROSIZE["Zero runtime cost"]
    FCOPY["F: Copy enables efficient flag passing"]
    BCLONE["B: Clone enables area splitting"]
end
subgraph subGraph2["Trait Bounds Enforcement"]
    PHANTOM["PhantomData<(F,P)>"]
    FCOPY["F: Copy enables efficient flag passing"]
    BCLONE["B: Clone enables area splitting"]
    BMAPPING["B: MappingBackend ensures consistent interface"]
end

B --> CLONE
B --> MABACKEND
F --> MAFLAG
F --> MBFLAG
P --> MBPT
PHANTOM --> TYPEREL
PHANTOM --> ZEROSIZE
```

The `PhantomData<(F,P)>` field in `MemoryArea` ensures that the compiler tracks the relationship between flag type `F` and page table type `P` even though `P` is not directly stored in the struct.

**Sources:** [src/area.rs(L29 - L34)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs#L29-L34) [src/area.rs(L15 - L22)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs#L15-L22)

## Error Handling Strategy

The crate implements a consistent error handling strategy that propagates failures through the operation chain while maintaining transactional semantics.

### Error Propagation Pattern

```mermaid
flowchart TD
subgraph subGraph3["Error Handling Locations"]
    AREAOPS["MemoryArea operations"]
    SETOPS["MemorySet operations"]
    PUBLICAPI["Public API boundary"]
end
subgraph subGraph2["Error Propagation"]
    PROPAGATE["? operator propagation"]
end
subgraph subGraph1["Error Sources"]
    BACKEND["Backend operation failure"]
    VALIDATION["Parameter validation"]
    OVERLAP["Overlap detection"]
end
subgraph subGraph0["Error Type Hierarchy"]
    MR["MappingResult = Result"]
    ME["MappingError"]
    BE["BadState"]
    IE["InvalidParam"]
    AE["AlreadyExists"]
end

AE --> PROPAGATE
AREAOPS --> SETOPS
BACKEND --> BE
BE --> PROPAGATE
IE --> PROPAGATE
ME --> AE
ME --> BE
ME --> IE
OVERLAP --> AE
PROPAGATE --> AREAOPS
SETOPS --> PUBLICAPI
VALIDATION --> IE
```

The error handling design ensures that failures at any level (backend, area, or set) are properly propagated to the caller with meaningful error information.

**Sources:** [src/area.rs(L6)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs#L6-L6) [src/area.rs(L90 - L110)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs#L90-L110) [src/set.rs(L98 - L114)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L98-L114)

### Backend Error Translation

The system translates boolean failure indicators from backends into structured error types:

```mermaid
sequenceDiagram
    participant MemoryArea as "MemoryArea"
    participant MappingBackend as "MappingBackend"
    participant MappingResult as "MappingResult"

    Note over MemoryArea,MappingResult: Backend Error Translation Pattern
    MemoryArea ->> MappingBackend: map(start, size, flags, page_table)
    MappingBackend ->> MemoryArea: return bool (success/failure)
    alt Backend returns true
    MemoryArea ->> MappingResult: Ok(())
    else Backend returns false
    MemoryArea ->> MappingResult: Err(MappingError::BadState)
    end
    Note over MemoryArea,MappingResult: Pattern used in map_area(), unmap_area(), protect_area()
```

This translation happens in the `MemoryArea` methods that interface with the backend, converting the simple boolean results into the richer `MappingResult` type for higher-level error handling.

**Sources:** [src/area.rs(L90 - L103)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs#L90-L103) [src/area.rs(L116 - L139)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/area.rs#L116-L139)