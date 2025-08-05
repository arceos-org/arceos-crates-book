# Usage and Examples

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md)
> * [src/tests.rs](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/tests.rs)

This document provides practical guidance for using the `memory_set` crate, including common usage patterns, complete working examples, and best practices. The examples demonstrate how to create memory mappings, manage overlapping areas, and implement custom backends for different page table systems.

For detailed implementation specifics of the core components, see [Implementation Details](/arceos-org/memory_set/2-implementation-details). For information about project setup and dependencies, see [Development and Project Setup](/arceos-org/memory_set/4-development-and-project-setup).

## Quick Start Example

The most basic usage involves creating a `MemorySet` with a custom backend implementation and using it to manage memory mappings. The following example from the README demonstrates the fundamental operations:

```mermaid
flowchart TD
subgraph subGraph1["Required Components"]
    Backend["MockBackend: MappingBackend"]
    PT["MockPageTable: [u8; MAX_ADDR]"]
    Flags["MockFlags: u8"]
    Area["MemoryArea::new(addr, size, flags, backend)"]
end
subgraph subGraph0["Basic Usage Flow"]
    Create["MemorySet::new()"]
    Map["map(MemoryArea, PageTable, unmap_overlap)"]
    Unmap["unmap(start, size, PageTable)"]
    Query["find(addr) / iter()"]
end

Area --> Map
Backend --> Map
Create --> Map
Flags --> Area
Map --> Unmap
PT --> Map
Unmap --> Query
```

**Basic Usage Pattern Flow**

Sources: [README.md(L18 - L88)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L18-L88)

The core workflow requires implementing the `MappingBackend` trait for your specific page table system, then using `MemorySet` to manage collections of `MemoryArea` objects. Here's the essential pattern from [README.md(L33 - L48)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L33-L48):

|Operation|Purpose|Key Parameters|
| --- | --- | --- |
|MemorySet::new()|Initialize empty memory set|None|
|map()|Add new memory mapping|MemoryArea, page table, overlap handling|
|unmap()|Remove memory mapping|Start address, size, page table|
|find()|Locate area containing address|Virtual address|
|iter()|Enumerate all areas|None|

## Backend Implementation Pattern

Every usage of `memory_set` requires implementing the `MappingBackend` trait. The implementation defines how memory operations interact with your specific page table system:

```mermaid
flowchart TD
subgraph subGraph2["Return Values"]
    Success["true: Operation succeeded"]
    Failure["false: Operation failed"]
end
subgraph subGraph1["Page Table Operations"]
    SetEntries["Set page table entries"]
    ClearEntries["Clear page table entries"]
    UpdateEntries["Update entry flags"]
end
subgraph subGraph0["MappingBackend Implementation"]
    Map["map(start, size, flags, pt)"]
    Unmap["unmap(start, size, pt)"]
    Protect["protect(start, size, new_flags, pt)"]
end

ClearEntries --> Failure
ClearEntries --> Success
Map --> SetEntries
Protect --> UpdateEntries
SetEntries --> Failure
SetEntries --> Success
Unmap --> ClearEntries
UpdateEntries --> Failure
UpdateEntries --> Success
```

**Backend Implementation Requirements**

Sources: [README.md(L51 - L87)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L51-L87) [src/tests.rs(L15 - L51)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/tests.rs#L15-L51)

The backend implementation handles the actual page table manipulation. Each method returns `bool` indicating success or failure:

* `map()`: Sets page table entries for the specified range if all entries are currently unmapped
* `unmap()`: Clears page table entries for the specified range if all entries are currently mapped
* `protect()`: Updates flags for page table entries in the specified range if all entries are currently mapped

## Memory Area Management Operations

The `MemorySet` provides sophisticated area management that handles overlaps, splits areas when needed, and maintains sorted order for efficient operations:

```mermaid
sequenceDiagram
    participant Client as Client
    participant MemorySet as MemorySet
    participant BTreeMap as BTreeMap
    participant MemoryArea as MemoryArea
    participant Backend as Backend

    Note over Client,Backend: Mapping with Overlap Resolution
    Client ->> MemorySet: "map(area, pt, unmap_overlap=true)"
    MemorySet ->> BTreeMap: "Check for overlapping areas"
    BTreeMap -->> MemorySet: "Found overlaps"
    MemorySet ->> MemorySet: "Remove/split overlapping areas"
    MemorySet ->> Backend: "unmap() existing regions"
    MemorySet ->> Backend: "map() new region"
    Backend -->> MemorySet: "Success"
    MemorySet ->> BTreeMap: "Insert new area"
    Note over Client,Backend: Unmapping that Splits Areas
    Client ->> MemorySet: "unmap(start, size, pt)"
    MemorySet ->> BTreeMap: "Find affected areas"
    loop "For each affected area"
        MemorySet ->> MemoryArea: "Check overlap relationship"
    alt "Partially overlapping"
        MemorySet ->> MemoryArea: "split() at boundaries"
        MemoryArea -->> MemorySet: "New area created"
    end
    MemorySet ->> Backend: "unmap() region"
    end
    MemorySet ->> BTreeMap: "Update area collection"
```

**Complex Memory Operations Sequence**

Sources: [src/tests.rs(L152 - L226)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/tests.rs#L152-L226) [README.md(L42 - L48)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L42-L48)

## Common Usage Patterns

Based on the test examples, several common patterns emerge for different memory management scenarios:

### Pattern 1: Basic Mapping and Unmapping

[src/tests.rs(L84 - L104)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/tests.rs#L84-L104) demonstrates mapping non-overlapping areas in a regular pattern, followed by querying operations.

### Pattern 2: Overlap Resolution

[src/tests.rs(L114 - L127)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/tests.rs#L114-L127) shows how to handle mapping conflicts using the `unmap_overlap` parameter to automatically resolve overlaps.

### Pattern 3: Area Splitting

[src/tests.rs(L166 - L192)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/tests.rs#L166-L192) illustrates how unmapping operations can split existing areas into multiple smaller areas when the unmapped region falls within an existing area.

### Pattern 4: Protection Changes

[src/tests.rs(L252 - L285)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/tests.rs#L252-L285) demonstrates using `protect()` operations to change memory permissions, which can also result in area splitting when applied to partial ranges.

## Error Handling Patterns

The crate uses `MappingResult<T>` for error handling, with `MappingError` variants indicating different failure modes:

|Error Type|Cause|Common Resolution|
| --- | --- | --- |
|AlreadyExists|Mapping overlaps existing area|Useunmap_overlap=trueor unmap manually|
|Backend failures|Page table operation failed|Check page table state and permissions|

Sources: [src/tests.rs(L114 - L121)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/tests.rs#L114-L121) [src/tests.rs(L53 - L66)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/tests.rs#L53-L66)

## Type System Usage

The generic type system allows customization for different environments:

```mermaid
flowchart TD
subgraph subGraph2["Instantiated Types"]
    MS["MemorySet"]
    MA["MemoryArea"]
end
subgraph subGraph1["Concrete Example Types"]
    MockFlags["MockFlags = u8"]
    MockPT["MockPageTable = [u8; MAX_ADDR]"]
    MockBackend["MockBackend struct"]
end
subgraph subGraph0["Generic Parameters"]
    F["F: Copy (Flags Type)"]
    PT["PT (Page Table Type)"]
    B["B: MappingBackend"]
end

B --> MockBackend
F --> MockFlags
MockBackend --> MA
MockBackend --> MS
MockFlags --> MA
MockFlags --> MS
MockPT --> MS
PT --> MockPT
```

**Type System Instantiation Pattern**

Sources: [README.md(L22 - L34)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L22-L34) [src/tests.rs(L5 - L13)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/src/tests.rs#L5-L13)

The examples use simple types (`u8` for flags, byte array for page table) but the system supports any types that meet the trait bounds. This flexibility allows integration with different operating system kernels, hypervisors, or memory management systems that have varying page table structures and flag representations.