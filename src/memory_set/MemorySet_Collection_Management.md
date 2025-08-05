# MemorySet Collection Management

> **Relevant source files**
> * [src/set.rs](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs)

This document covers the internal collection management mechanisms of `MemorySet`, focusing on how it organizes, tracks, and manipulates collections of memory areas using a `BTreeMap` data structure. It examines overlap detection algorithms, area lifecycle operations, and complex manipulation procedures like splitting and shrinking.

For information about individual `MemoryArea` objects and the `MappingBackend` trait, see [MemoryArea and MappingBackend](/arceos-org/memory_set/2.1-memoryarea-and-mappingbackend). For the public API interface, see [Public API and Error Handling](/arceos-org/memory_set/2.3-public-api-and-error-handling).

## Core Data Structure Organization

The `MemorySet` struct uses a `BTreeMap<VirtAddr, MemoryArea<F, P, B>>` as its primary storage mechanism, where the key is the starting virtual address of each memory area. This design provides efficient logarithmic-time operations for address-based queries and maintains areas in sorted order by their start addresses.

**MemorySet BTreeMap Organization**

```mermaid
flowchart TD
subgraph subGraph2["Key Properties"]
    Sorted["Keys sorted by address"]
    Unique["Each key is area.start()"]
    NoOverlap["No overlapping ranges"]
end
subgraph subGraph1["BTreeMap Structure"]
    Key1["va!(0x1000)"]
    Key2["va!(0x3000)"]
    Key3["va!(0x5000)"]
    Area1["MemoryArea[0x1000-0x2000]"]
    Area2["MemoryArea[0x3000-0x4000]"]
    Area3["MemoryArea[0x5000-0x6000]"]
end
subgraph MemorySet["MemorySet<F,P,B>"]
    BTree["BTreeMap<VirtAddr, MemoryArea>"]
end

BTree --> Key1
BTree --> Key2
BTree --> Key3
BTree --> NoOverlap
BTree --> Sorted
BTree --> Unique
Key1 --> Area1
Key2 --> Area2
Key3 --> Area3
```

**Sources:** [src/set.rs(L9 - L11)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L9-L11)

## Basic Collection Operations

The `MemorySet` provides fundamental collection operations that leverage the `BTreeMap`'s sorted structure:

|Operation|Method|Time Complexity|Purpose|
| --- | --- | --- | --- |
|Count areas|len()|O(1)|Returns number of areas|
|Check emptiness|is_empty()|O(1)|Tests if collection is empty|
|Iterate areas|iter()|O(n)|Provides iterator over all areas|
|Find by address|find(addr)|O(log n)|Locates area containing address|

The `find()` method demonstrates efficient address lookup using range queries on the `BTreeMap`:

**Address Lookup Algorithm**

```mermaid
flowchart TD
Start["find(addr)"]
RangeQuery["areas.range(..=addr).last()"]
GetCandidate["Get last area with start ≤ addr"]
CheckContains["Check if area.va_range().contains(addr)"]
ReturnArea["Return Some(area)"]
ReturnNone["Return None"]

CheckContains --> ReturnArea
CheckContains --> ReturnNone
GetCandidate --> CheckContains
RangeQuery --> GetCandidate
Start --> RangeQuery
```

**Sources:** [src/set.rs(L21 - L55)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L21-L55)

## Overlap Detection and Management

The `overlaps()` method implements efficient overlap detection by checking at most two adjacent areas in the sorted collection, rather than scanning all areas:

**Overlap Detection Strategy**

```mermaid
flowchart TD
CheckOverlap["overlaps(range)"]
CheckBefore["Check area before range.start"]
CheckAfter["Check area at/after range.start"]
BeforeRange["areas.range(..range.start).last()"]
AfterRange["areas.range(range.start..).next()"]
TestBeforeOverlap["before.va_range().overlaps(range)"]
TestAfterOverlap["after.va_range().overlaps(range)"]
ReturnTrue1["Return true"]
ReturnTrue2["Return true"]
ContinueAfter["Continue to after check"]
ReturnFalse["Return false"]

AfterRange --> TestAfterOverlap
BeforeRange --> TestBeforeOverlap
CheckAfter --> AfterRange
CheckBefore --> BeforeRange
CheckOverlap --> CheckAfter
CheckOverlap --> CheckBefore
TestAfterOverlap --> ReturnFalse
TestAfterOverlap --> ReturnTrue2
TestBeforeOverlap --> ContinueAfter
TestBeforeOverlap --> ReturnTrue1
```

This algorithm achieves O(log n) complexity by exploiting the sorted nature of the `BTreeMap` and the non-overlapping invariant of stored areas.

**Sources:** [src/set.rs(L36 - L49)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L36-L49)

## Memory Area Addition and Conflict Resolution

The `map()` operation handles the complex process of adding new areas while managing potential conflicts:

**Map Operation Flow**

```mermaid
sequenceDiagram
    participant Client as Client
    participant MemorySet as MemorySet
    participant BTreeMap as BTreeMap
    participant MemoryArea as MemoryArea

    Client ->> MemorySet: "map(area, page_table, unmap_overlap)"
    MemorySet ->> MemorySet: "Validate area.va_range().is_empty()"
    MemorySet ->> MemorySet: "overlaps(area.va_range())"
    alt "Overlap detected"
    alt "unmap_overlap = true"
        MemorySet ->> MemorySet: "unmap(area.start(), area.size(), page_table)"
        Note over MemorySet: "Remove conflicting areas"
    else "unmap_overlap = false"
        MemorySet -->> Client: "MappingError::AlreadyExists"
    end
    end
    MemorySet ->> MemoryArea: "area.map_area(page_table)"
    MemoryArea -->> MemorySet: "MappingResult"
    MemorySet ->> BTreeMap: "insert(area.start(), area)"
    MemorySet -->> Client: "Success"
```

**Sources:** [src/set.rs(L85 - L114)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L85-L114)

## Complex Area Manipulation Operations

### Unmap Operation with Area Splitting

The `unmap()` operation implements sophisticated area manipulation, including splitting areas when the unmapped region falls in the middle of an existing area:

**Unmap Cases and Transformations**

```mermaid
flowchart TD
subgraph subGraph3["Case 4: Middle Split"]
    subgraph subGraph1["Case 2: Right Boundary Intersection"]
        C4Before["[████████████████][unmap]"]
        C4After1["[██]      [██]split into two"]
        C3Before["[████████████][unmap]"]
        C3After1["[████]shrink_left"]
        C2Before["[██████████████][unmap]"]
        C2After1["[████]shrink_right"]
        C1Before["[████████████]Existing Area"]
        C1After1["(removed)"]
        subgraph subGraph2["Case 3: Left Boundary Intersection"]
            subgraph subGraph0["Case 1: Full Containment"]
                C4Before["[████████████████][unmap]"]
                C4After1["[██]      [██]split into two"]
                C3Before["[████████████][unmap]"]
                C3After1["[████]shrink_left"]
                C2Before["[██████████████][unmap]"]
                C2After1["[████]shrink_right"]
                C1Before["[████████████]Existing Area"]
                C1After1["(removed)"]
            end
        end
    end
end

C1Before --> C1After1
C2Before --> C2After1
C3Before --> C3After1
C4Before --> C4After1
```

The implementation handles these cases through a three-phase process:

1. **Full Removal**: Remove areas completely contained in the unmap range using `retain()`
2. **Left Boundary Processing**: Shrink or split areas that intersect the left boundary
3. **Right Boundary Processing**: Shrink areas that intersect the right boundary

**Sources:** [src/set.rs(L116 - L169)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L116-L169)

### Protect Operation with Multi-way Splitting

The `protect()` operation changes memory flags within a range and may require splitting existing areas into multiple parts:

**Protect Operation Area Handling**

```

```

**Sources:** [src/set.rs(L180 - L247)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L180-L247)

## Free Space Management

The `find_free_area()` method locates unallocated address ranges by examining gaps between consecutive areas:

**Free Area Search Algorithm**

```mermaid
flowchart TD
FindFree["find_free_area(hint, size, limit)"]
InitLastEnd["last_end = max(hint, limit.start)"]
IterateAreas["For each (addr, area) in areas"]
CheckGap["if last_end + size <= addr"]
ReturnGap["Return Some(last_end)"]
UpdateEnd["last_end = area.end()"]
NextArea["Continue to next area"]
NoMoreAreas["End of areas reached"]
CheckFinalGap["if last_end + size <= limit.end"]
ReturnFinal["Return Some(last_end)"]
ReturnNone["Return None"]

CheckFinalGap --> ReturnFinal
CheckFinalGap --> ReturnNone
CheckGap --> ReturnGap
CheckGap --> UpdateEnd
FindFree --> InitLastEnd
InitLastEnd --> IterateAreas
IterateAreas --> CheckGap
IterateAreas --> NoMoreAreas
NextArea --> IterateAreas
NoMoreAreas --> CheckFinalGap
UpdateEnd --> NextArea
```

**Sources:** [src/set.rs(L57 - L83)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L57-L83)

## Collection Cleanup Operations

The `clear()` method provides atomic cleanup of all memory areas and their underlying mappings:

```python
// Unmaps all areas from the page table, then clears the collection
pub fn clear(&mut self, page_table: &mut P) -> MappingResult
```

This operation iterates through all areas, calls `unmap_area()` on each, and then clears the `BTreeMap` if all unmapping operations succeed.

**Sources:** [src/set.rs(L171 - L178)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/set.rs#L171-L178)