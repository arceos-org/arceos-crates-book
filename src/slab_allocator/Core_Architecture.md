# Core Architecture

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs)
> * [src/slab.rs](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/slab.rs)

This document provides a comprehensive overview of the slab allocator's hybrid memory management architecture. It explains the fundamental design decisions, component interactions, and allocation strategies that enable efficient memory management in `no_std` environments. For detailed implementation specifics of individual components, see [Heap Allocator Design](/arceos-org/slab_allocator/3.1-heap-allocator-design) and [Slab Implementation](/arceos-org/slab_allocator/3.2-slab-implementation).

## Hybrid Allocation Strategy

The slab allocator implements a two-tier memory allocation system that optimizes for both small fixed-size allocations and large variable-size allocations. The core design principle is to route allocation requests to the most appropriate allocator based on size and alignment requirements.

### Allocation Decision Tree

```mermaid
flowchart TD
A["Layout::size() > 4096"]
B["Size Check"]
C["buddy_system_allocator::Heap<32>"]
D["Size and Alignment Analysis"]
E["size <= 64 && align <= 64"]
F["Slab<64>"]
G["size <= 128 && align <= 128"]
H["Slab<128>"]
I["size <= 256 && align <= 256"]
J["Slab<256>"]
K["size <= 512 && align <= 512"]
L["Slab<512>"]
M["size <= 1024 && align <= 1024"]
N["Slab<1024>"]
O["size <= 2048 && align <= 2048"]
P["Slab<2048>"]
Q["Slab<4096>"]

A --> B
B --> C
B --> D
D --> E
E --> F
E --> G
G --> H
G --> I
I --> J
I --> K
K --> L
K --> M
M --> N
M --> O
O --> P
O --> Q
```

Sources: [src/lib.rs(L207 - L226)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L207-L226)

### Component Architecture

The `Heap` struct orchestrates multiple specialized allocators to provide a unified memory management interface:

```mermaid
flowchart TD
subgraph subGraph1["HeapAllocator Enum"]
    J["Slab64Bytes"]
    K["Slab128Bytes"]
    L["Slab256Bytes"]
    M["Slab512Bytes"]
    N["Slab1024Bytes"]
    O["Slab2048Bytes"]
    P["Slab4096Bytes"]
    Q["BuddyAllocator"]
end
subgraph subGraph0["Heap Struct"]
    A["pub struct Heap"]
    B["slab_64_bytes: Slab<64>"]
    C["slab_128_bytes: Slab<128>"]
    D["slab_256_bytes: Slab<256>"]
    E["slab_512_bytes: Slab<512>"]
    F["slab_1024_bytes: Slab<1024>"]
    G["slab_2048_bytes: Slab<2048>"]
    H["slab_4096_bytes: Slab<4096>"]
    I["buddy_allocator: buddy_system_allocator::Heap<32>"]
end
R["layout_to_allocator()"]

A --> B
A --> C
A --> D
A --> E
A --> F
A --> G
A --> H
A --> I
J --> B
K --> C
L --> D
M --> E
N --> F
O --> G
P --> H
Q --> I
R --> J
R --> K
R --> L
R --> M
R --> N
R --> O
R --> P
R --> Q
```

Sources: [src/lib.rs(L40 - L49)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L40-L49) [src/lib.rs(L27 - L36)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L27-L36)

## Memory Organization and Block Management

Each slab allocator manages fixed-size blocks through a linked list of free blocks. The system uses a hybrid approach where slabs can dynamically grow by requesting memory from the buddy allocator.

### Slab Internal Structure

```mermaid
flowchart TD
subgraph subGraph1["FreeBlockList Operations"]
    E["pop()"]
    F["Returns available block"]
    G["push()"]
    H["Adds freed block to list"]
    I["new()"]
    J["Initializes from memory range"]
end
subgraph Slab<BLK_SIZE>["Slab"]
    A["free_block_list: FreeBlockList"]
    B["head: Option<&'static mut FreeBlock>"]
    C["len: usize"]
    D["total_blocks: usize"]
end
subgraph subGraph2["Dynamic Growth"]
    K["allocate() fails"]
    L["Request SET_SIZE * BLK_SIZE from buddy"]
    M["grow()"]
    N["Create new FreeBlockList"]
    O["Transfer blocks to main list"]
end

A --> B
A --> C
A --> E
A --> G
B --> F
B --> H
E --> F
G --> H
I --> J
K --> L
L --> M
M --> N
N --> O
```

Sources: [src/slab.rs(L4 - L7)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/slab.rs#L4-L7) [src/slab.rs(L65 - L68)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/slab.rs#L65-L68)

## Allocation Flow and Integration Points

The allocation process demonstrates the tight integration between slab allocators and the buddy allocator, with automatic fallback and growth mechanisms.

### Allocation Process Flow

```mermaid
sequenceDiagram
    participant Client as Client
    participant Heap as Heap
    participant Slab as Slab
    participant FreeBlockList as FreeBlockList
    participant BuddyAllocator as BuddyAllocator

    Client ->> Heap: allocate(layout)
    Heap ->> Heap: layout_to_allocator(layout)
    alt Size <= 4096
        Heap ->> Slab: allocate(layout, buddy)
        Slab ->> FreeBlockList: pop()
    alt Free block available
        FreeBlockList -->> Slab: Some(block)
        Slab -->> Heap: Ok(block.addr())
    else No free blocks
        Slab ->> BuddyAllocator: alloc(SET_SIZE * BLK_SIZE)
        BuddyAllocator -->> Slab: Ok(ptr)
        Slab ->> Slab: grow(ptr, size)
        Slab ->> FreeBlockList: pop()
        FreeBlockList -->> Slab: Some(block)
        Slab -->> Heap: Ok(block.addr())
    end
    Heap -->> Client: Ok(address)
    else Size > 4096
        Heap ->> BuddyAllocator: alloc(layout)
        BuddyAllocator -->> Heap: Ok(ptr)
        Heap -->> Client: Ok(address)
    end
```

Sources: [src/lib.rs(L135 - L164)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L135-L164) [src/slab.rs(L35 - L55)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/slab.rs#L35-L55)

## Performance Characteristics and Design Rationale

### Allocation Complexity

|Allocation Type|Time Complexity|Space Overhead|Use Case|
| --- | --- | --- | --- |
|Slab (≤ 4096 bytes)|O(1)|Fixed per slab|Frequent small allocations|
|Buddy (> 4096 bytes)|O(log n)|Variable|Large or variable-size allocations|
|Slab growth|O(SET_SIZE)|Batch allocation|Slab expansion|

### Key Design Constants

```javascript
const SET_SIZE: usize = 64;           // Blocks per growth operation
const MIN_HEAP_SIZE: usize = 0x8000;  // Minimum heap size (32KB)
```

The `SET_SIZE` constant controls the granularity of slab growth operations. When a slab runs out of free blocks, it requests `SET_SIZE * BLK_SIZE` bytes from the buddy allocator, ensuring efficient batch allocation while minimizing buddy allocator overhead.

Sources: [src/lib.rs(L24 - L25)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L24-L25) [src/slab.rs(L44 - L48)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/slab.rs#L44-L48)

## Memory Statistics and Monitoring

The architecture provides comprehensive memory usage tracking across both allocation subsystems:

### Statistics Calculation

```mermaid
flowchart TD
subgraph subGraph1["Per-Slab Calculation"]
    I["slab.total_blocks() * BLOCK_SIZE"]
    J["slab.used_blocks() * BLOCK_SIZE"]
end
subgraph subGraph0["Memory Metrics"]
    A["total_bytes()"]
    B["Sum of all slab capacities"]
    C["buddy_allocator.stats_total_bytes()"]
    D["used_bytes()"]
    E["Sum of allocated slab blocks"]
    F["buddy_allocator.stats_alloc_actual()"]
    G["available_bytes()"]
    H["total_bytes() - used_bytes()"]
end

A --> B
A --> C
B --> I
D --> E
D --> F
E --> J
G --> H
```

Sources: [src/lib.rs(L228 - L255)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L228-L255)

## Integration with Buddy Allocator

The system maintains a symbiotic relationship with the `buddy_system_allocator` crate, using it both as a fallback for large allocations and as a memory provider for slab growth operations.

### Buddy Allocator Usage Patterns

1. **Direct allocation**: Requests over 4096 bytes bypass slabs entirely
2. **Slab growth**: Slabs request memory chunks for expansion
3. **Memory initialization**: Initial heap setup uses buddy allocator
4. **Memory extension**: Additional memory regions added through buddy allocator

The buddy allocator is configured with a 32-level tree (`Heap<32>`), providing efficient allocation for a wide range of block sizes while maintaining reasonable memory overhead.

Sources: [src/lib.rs(L48)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L48-L48) [src/lib.rs(L80 - L85)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L80-L85) [src/lib.rs(L158 - L163)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L158-L163)