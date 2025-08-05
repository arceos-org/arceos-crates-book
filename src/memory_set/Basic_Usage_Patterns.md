# Basic Usage Patterns

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md)

This document demonstrates the fundamental usage patterns for the memory_set crate, showing how to create and configure `MemorySet`, `MemoryArea`, and implement `MappingBackend` for typical memory management scenarios. The examples here focus on the core API usage and basic operations like mapping, unmapping, and memory protection.

For detailed implementation internals, see [Implementation Details](/arceos-org/memory_set/2-implementation-details). For advanced usage scenarios and testing patterns, see [Advanced Examples and Testing](/arceos-org/memory_set/3.2-advanced-examples-and-testing).

## Setting Up Types and Backend

The memory_set crate uses a generic type system that requires three type parameters to be specified. The typical pattern involves creating type aliases and implementing a backend that handles the actual memory operations.

### Type Configuration Pattern

|Component|Purpose|Example Type|
| --- | --- | --- |
|F(Flags)|Memory protection flags|u8, custom bitflags|
|PT(Page Table)|Page table representation|Array, actual page table struct|
|B(Backend)|Memory mapping implementation|Custom struct implementingMappingBackend|

The basic setup pattern follows this structure:

```
type MockFlags = u8;
type MockPageTable = [MockFlags; MAX_ADDR];
struct MockBackend;
```

**Core Type Relationships**

```mermaid
flowchart TD
subgraph subGraph2["Instantiated Types"]
    MSI["MemorySet"]
    MAI["MemoryArea"]
end
subgraph subGraph1["Generic Types"]
    MS["MemorySet"]
    MA["MemoryArea"]
    MB["MappingBackend trait"]
end
subgraph subGraph0["User Configuration"]
    UF["MockFlags = u8"]
    UPT["MockPageTable = [u8; MAX_ADDR]"]
    UB["MockBackend struct"]
end

MA --> MAI
MS --> MSI
MSI --> MAI
UB --> MA
UB --> MB
UB --> MS
UF --> MA
UF --> MS
UPT --> MS
```

Sources: [README.md(L22 - L31)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L22-L31)

## Creating and Initializing MemorySet

The basic pattern for creating a `MemorySet` involves calling the `new()` constructor with appropriate type parameters:

```javascript
let mut memory_set = MemorySet::<MockFlags, MockPageTable, MockBackend>::new();
```

The `MemorySet` maintains an internal `BTreeMap` that organizes memory areas by their starting virtual addresses, enabling efficient overlap detection and range queries.

**MemorySet Initialization Flow**

```mermaid
flowchart TD
subgraph subGraph0["Required Context"]
    PT["&mut MockPageTable"]
    AREAS["MemoryArea instances"]
end
NEW["MemorySet::new()"]
BT["BTreeMap"]
EMPTY["Empty memory set ready for mapping"]

BT --> EMPTY
EMPTY --> AREAS
EMPTY --> PT
NEW --> BT
```

Sources: [README.md(L34)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L34-L34)

## Mapping Memory Areas

The fundamental mapping operation involves creating a `MemoryArea` and adding it to the `MemorySet`. The pattern requires specifying the virtual address range, flags, and backend implementation.

### Basic Mapping Pattern

```yaml
memory_set.map(
    MemoryArea::new(va!(0x1000), 0x4000, flags, MockBackend),
    &mut page_table,
    unmap_overlap_flag,
)
```

### MemoryArea Construction Parameters

|Parameter|Type|Purpose|
| --- | --- | --- |
|start|VirtAddr|Starting virtual address|
|size|usize|Size in bytes|
|flags|F|Memory protection flags|
|backend|B|Backend implementation|

**Memory Mapping Operation Flow**

```mermaid
sequenceDiagram
    participant User as User
    participant MemorySet as MemorySet
    participant BTreeMap as BTreeMap
    participant MemoryArea as MemoryArea
    participant MappingBackend as MappingBackend

    User ->> MemorySet: "map(area, page_table, false)"
    MemorySet ->> BTreeMap: "Check for overlaps"
    alt No overlaps
        MemorySet ->> MemoryArea: "Get backend reference"
        MemoryArea ->> MappingBackend: "map(start, size, flags, pt)"
        MappingBackend -->> MemoryArea: "Success/failure"
        MemorySet ->> BTreeMap: "Insert area"
        MemorySet -->> User: "Ok(())"
    else Overlaps found and
        MemorySet -->> User: "Err(AlreadyExists)"
    end
```

Sources: [README.md(L36 - L41)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L36-L41)

## Unmapping and Area Management

Unmapping operations can result in area splitting when the unmapped region falls within an existing area's boundaries. This demonstrates the sophisticated area management capabilities.

### Unmapping Pattern

```
memory_set.unmap(start_addr, size, &mut page_table)
```

The unmap operation handles three scenarios:

* **Complete removal**: Area fully contained within unmap range
* **Area splitting**: Unmap range falls within area boundaries
* **Partial removal**: Unmap range overlaps area boundaries

**Area Splitting During Unmap**

```mermaid
flowchart TD
subgraph subGraph3["BTreeMap State"]
    MAP1["Key: 0x1000 → MemoryArea(0x1000-0x2000)"]
    MAP2["Key: 0x4000 → MemoryArea(0x4000-0x5000)"]
end
subgraph subGraph2["After Unmap"]
    LEFT["MemoryArea: 0x1000-0x2000"]
    RIGHT["MemoryArea: 0x4000-0x5000"]
end
subgraph subGraph1["Unmap Operation"]
    UNMAP["unmap(0x2000, 0x2000)"]
end
subgraph subGraph0["Before Unmap"]
    ORIG["MemoryArea: 0x1000-0x5000"]
end

LEFT --> MAP1
ORIG --> UNMAP
RIGHT --> MAP2
UNMAP --> LEFT
UNMAP --> RIGHT
```

Sources: [README.md(L42 - L48)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L42-L48)

## Implementing MappingBackend

The `MappingBackend` trait defines the interface for actual memory operations. Implementations must provide three core methods that manipulate the underlying page table or memory management structure.

### Required Methods

|Method|Purpose|Return Type|
| --- | --- | --- |
|map|Establish new mappings|bool(success/failure)|
|unmap|Remove existing mappings|bool(success/failure)|
|protect|Change mapping permissions|bool(success/failure)|

### Implementation Pattern

The typical pattern involves checking existing state and modifying page table entries:

```rust
impl MappingBackend<MockFlags, MockPageTable> for MockBackend {
    fn map(&self, start: VirtAddr, size: usize, flags: MockFlags, pt: &mut MockPageTable) -> bool {
        // Check for conflicts, then set entries
    }
    
    fn unmap(&self, start: VirtAddr, size: usize, pt: &mut MockPageTable) -> bool {
        // Verify mappings exist, then clear entries
    }
    
    fn protect(&self, start: VirtAddr, size: usize, new_flags: MockFlags, pt: &mut MockPageTable) -> bool {
        // Verify mappings exist, then update flags
    }
}
```

**MappingBackend Method Responsibilities**

```mermaid
flowchart TD
subgraph subGraph2["Implementation Details"]
    ITER["pt.iter_mut().skip(start).take(size)"]
    SET["*entry = flags"]
    CLEAR["*entry = 0"]
end
subgraph subGraph1["Page Table Operations"]
    CHECK["Validate existing state"]
    MODIFY["Modify page table entries"]
    VERIFY["Return success/failure"]
end
subgraph subGraph0["MappingBackend Interface"]
    MAP["map(start, size, flags, pt)"]
    UNMAP["unmap(start, size, pt)"]
    PROTECT["protect(start, size, flags, pt)"]
end

CHECK --> MODIFY
ITER --> CLEAR
ITER --> SET
MAP --> CHECK
MODIFY --> ITER
MODIFY --> VERIFY
PROTECT --> CHECK
UNMAP --> CHECK
```

Sources: [README.md(L51 - L87)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L51-L87)

## Complete Usage Flow

The following demonstrates a complete usage pattern that combines all the basic operations:

### Initialization and Setup

1. Define type aliases for flags, page table, and backend
2. Create page table instance
3. Initialize empty `MemorySet`

### Memory Operations

1. Create `MemoryArea` with desired parameters
2. Map area using `MemorySet::map()`
3. Perform unmapping operations that may split areas
4. Iterate over resulting areas to verify state

### Error Handling

All operations return `MappingResult<T>` which wraps either success values or `MappingError` variants for proper error handling.

**Complete Operation Sequence**

```mermaid
flowchart TD
subgraph subGraph0["Backend Operations"]
    BACKEND_MAP["backend.map()"]
    BACKEND_UNMAP["backend.unmap()"]
    PT_UPDATE["Page table updates"]
end
INIT["Initialize types and MemorySet"]
CREATE["Create MemoryArea"]
MAP["memory_set.map()"]
CHECK["Check result"]
UNMAP["memory_set.unmap()"]
HANDLE["Handle MappingError"]
VERIFY["Verify area splitting"]
ITER["memory_set.iter()"]
END["Operation complete"]

BACKEND_MAP --> PT_UPDATE
BACKEND_UNMAP --> PT_UPDATE
CHECK --> HANDLE
CHECK --> UNMAP
CREATE --> MAP
HANDLE --> END
INIT --> CREATE
ITER --> END
MAP --> BACKEND_MAP
MAP --> CHECK
UNMAP --> BACKEND_UNMAP
UNMAP --> VERIFY
VERIFY --> ITER
```

Sources: [README.md(L18 - L88)&emsp;](https://github.com/arceos-org/memory_set/blob/73b51e2b/README.md#L18-L88)