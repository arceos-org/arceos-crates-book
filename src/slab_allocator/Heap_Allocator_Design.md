# Heap Allocator Design

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs)

This document covers the design and implementation of the main `Heap` struct, which serves as the primary interface for memory allocation in the slab_allocator crate. It explains the hybrid allocation strategy, routing logic, and integration with the buddy system allocator. For details about individual slab implementations and their internal mechanics, see [Slab Implementation](/arceos-org/slab_allocator/3.2-slab-implementation). For API usage examples and method documentation, see [API Reference](/arceos-org/slab_allocator/4-api-reference).

## Core Components Overview

The heap allocator is built around the `Heap` struct, which coordinates between multiple fixed-size slab allocators and a buddy system allocator for larger requests.

### Heap Structure Components

```mermaid
flowchart TD
subgraph subGraph0["Heap Struct Fields"]
    C1["SET_SIZE: usize = 64"]
    C2["MIN_HEAP_SIZE: usize = 0x8000"]
    B["buddy_allocator: buddy_system_allocator::Heap<32>"]
    subgraph subGraph1["HeapAllocator Enum Variants"]
        E["HeapAllocator"]
        E64["Slab64Bytes"]
        E128["Slab128Bytes"]
        E256["Slab256Bytes"]
        E512["Slab512Bytes"]
        E1024["Slab1024Bytes"]
        E2048["Slab2048Bytes"]
        E4096["Slab4096Bytes"]
        EB["BuddyAllocator"]
        H["Heap"]
        S64["slab_64_bytes: Slab<64>"]
        S128["slab_128_bytes: Slab<128>"]
        S256["slab_256_bytes: Slab<256>"]
        S512["slab_512_bytes: Slab<512>"]
        S1024["slab_1024_bytes: Slab<1024>"]
        S2048["slab_2048_bytes: Slab<2048>"]
        S4096["slab_4096_bytes: Slab<4096>"]
        subgraph Constants["Constants"]
            C1["SET_SIZE: usize = 64"]
            C2["MIN_HEAP_SIZE: usize = 0x8000"]
        end
    end
end

E --> E1024
E --> E128
E --> E2048
E --> E256
E --> E4096
E --> E512
E --> E64
E --> EB
H --> B
H --> S1024
H --> S128
H --> S2048
H --> S256
H --> S4096
H --> S512
H --> S64
```

**Sources:** [src/lib.rs(L40 - L49)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L40-L49) [src/lib.rs(L27 - L36)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L27-L36) [src/lib.rs(L24 - L25)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L24-L25)

The `Heap` struct contains seven slab allocators, each handling fixed-size blocks, plus a `buddy_system_allocator::Heap` for variable-size allocations. The `HeapAllocator` enum provides a type-safe way to route allocation requests to the appropriate allocator.

## Allocation Routing Strategy

The heart of the heap allocator's design is the `layout_to_allocator` function, which determines which allocator to use based on the requested layout's size and alignment requirements.

### Layout Routing Logic

```mermaid
flowchart TD
L["Layout { size, align }"]
Check1["size > 4096?"]
BA["HeapAllocator::BuddyAllocator"]
Check64["size ≤ 64 && align ≤ 64?"]
S64["HeapAllocator::Slab64Bytes"]
Check128["size ≤ 128 && align ≤ 128?"]
S128["HeapAllocator::Slab128Bytes"]
Check256["size ≤ 256 && align ≤ 256?"]
S256["HeapAllocator::Slab256Bytes"]
Check512["size ≤ 512 && align ≤ 512?"]
S512["HeapAllocator::Slab512Bytes"]
Check1024["size ≤ 1024 && align ≤ 1024?"]
S1024["HeapAllocator::Slab1024Bytes"]
Check2048["size ≤ 2048 && align ≤ 2048?"]
S2048["HeapAllocator::Slab2048Bytes"]
S4096["HeapAllocator::Slab4096Bytes"]

Check1 --> BA
Check1 --> Check64
Check1024 --> Check2048
Check1024 --> S1024
Check128 --> Check256
Check128 --> S128
Check2048 --> S2048
Check2048 --> S4096
Check256 --> Check512
Check256 --> S256
Check512 --> Check1024
Check512 --> S512
Check64 --> Check128
Check64 --> S64
L --> Check1
```

**Sources:** [src/lib.rs(L208 - L226)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L208-L226)

The routing logic follows a cascading pattern where both size and alignment constraints must be satisfied. This ensures that allocations are placed in the smallest slab that can accommodate both the requested size and alignment requirements.

## Memory Management Workflow

The allocation and deallocation processes follow a consistent pattern of routing through the `HeapAllocator` enum to the appropriate underlying allocator.

### Allocation Process Flow

```mermaid
sequenceDiagram
    participant ClientCode as "Client Code"
    participant Heapallocate as "Heap::allocate"
    participant layout_to_allocator as "layout_to_allocator"
    participant Slaballocate as "Slab::allocate"
    participant buddy_allocator as "buddy_allocator"

    ClientCode ->> Heapallocate: "allocate(layout)"
    Heapallocate ->> layout_to_allocator: "layout_to_allocator(&layout)"
    layout_to_allocator -->> Heapallocate: "HeapAllocator variant"
    alt "Slab allocation"
        Heapallocate ->> Slaballocate: "slab.allocate(layout, &mut buddy_allocator)"
        Slaballocate ->> Slaballocate: "Check free blocks"
    alt "No free blocks"
        Slaballocate ->> buddy_allocator: "Request memory for growth"
        buddy_allocator -->> Slaballocate: "Memory region"
        Slaballocate ->> Slaballocate: "Initialize new blocks"
    end
    Slaballocate -->> Heapallocate: "Result<usize, AllocError>"
    else "Buddy allocation"
        Heapallocate ->> buddy_allocator: "buddy_allocator.alloc(layout)"
        buddy_allocator -->> Heapallocate: "Result<NonNull<u8>, AllocError>"
    end
    Heapallocate -->> ClientCode: "Result<usize, AllocError>"
```

**Sources:** [src/lib.rs(L135 - L164)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L135-L164) [src/lib.rs(L177 - L190)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L177-L190)

### Deallocation Process Flow

```mermaid
sequenceDiagram
    participant ClientCode as "Client Code"
    participant Heapdeallocate as "Heap::deallocate"
    participant layout_to_allocator as "layout_to_allocator"
    participant Slabdeallocate as "Slab::deallocate"
    participant buddy_allocator as "buddy_allocator"

    ClientCode ->> Heapdeallocate: "deallocate(ptr, layout)"
    Heapdeallocate ->> layout_to_allocator: "layout_to_allocator(&layout)"
    layout_to_allocator -->> Heapdeallocate: "HeapAllocator variant"
    alt "Slab deallocation"
        Heapdeallocate ->> Slabdeallocate: "slab.deallocate(ptr)"
        Slabdeallocate ->> Slabdeallocate: "Add block to free list"
        Slabdeallocate -->> Heapdeallocate: "void"
    else "Buddy deallocation"
        Heapdeallocate ->> buddy_allocator: "buddy_allocator.dealloc(ptr, layout)"
        buddy_allocator -->> Heapdeallocate: "void"
    end
```

**Sources:** [src/lib.rs(L177 - L190)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L177-L190)

## Performance Characteristics

The hybrid design provides different performance guarantees depending on allocation size:

|Allocation Size|Allocator Used|Time Complexity|Characteristics|
| --- | --- | --- | --- |
|≤ 4096 bytes|Slab allocator|O(1)|Fixed-size blocks, predictable performance|
|> 4096 bytes|Buddy allocator|O(n)|Variable-size blocks, flexible but slower|

**Sources:** [src/lib.rs(L134 - L135)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L134-L135) [src/lib.rs(L172 - L173)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L172-L173)

The O(1) performance for small allocations is achieved because slab allocators maintain free lists of pre-allocated blocks. The O(n) complexity for large allocations reflects the buddy allocator's tree-based search for appropriately sized free blocks.

## Memory Statistics and Monitoring

The `Heap` struct provides comprehensive memory usage statistics by aggregating data from all component allocators.

### Statistics Methods

```mermaid
flowchart TD
subgraph subGraph0["Memory Statistics API"]
    TB["total_bytes()"]
    Calc1["Sum of slab total_blocks() * block_size"]
    Calc2["Unsupported markdown: list"]
    UB["used_bytes()"]
    Calc3["Sum of slab used_blocks() * block_size"]
    Calc4["Unsupported markdown: list"]
    AB["available_bytes()"]
    Calc5["total_bytes() - used_bytes()"]
    US["usable_size(layout)"]
    Route["Route by HeapAllocator"]
    SlabSize["Slab: (layout.size(), slab_size)"]
    BuddySize["Buddy: (layout.size(), layout.size())"]
end

AB --> Calc5
Route --> BuddySize
Route --> SlabSize
TB --> Calc1
TB --> Calc2
UB --> Calc3
UB --> Calc4
US --> Route
```

**Sources:** [src/lib.rs(L229 - L255)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L229-L255) [src/lib.rs(L194 - L205)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L194-L205)

The statistics methods provide real-time visibility into memory usage across both slab and buddy allocators, enabling monitoring and debugging of allocation patterns.

## Integration Points

The heap allocator serves as a coordinator between the slab allocators and the buddy system allocator, handling several key integration scenarios:

### Slab Growth Mechanism

When a slab runs out of free blocks, it requests additional memory from the buddy allocator through the `Slab::allocate` method. This allows slabs to grow dynamically based on demand while maintaining their fixed-block-size efficiency.

### Memory Addition

The `add_memory` and `_grow` methods allow runtime expansion of the heap, with new memory being routed to the appropriate allocator:

**Sources:** [src/lib.rs(L95 - L106)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L95-L106) [src/lib.rs(L116 - L129)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L116-L129)

* Public `add_memory`: Adds memory directly to the buddy allocator
* Private `_grow`: Routes memory to specific slabs based on the `HeapAllocator` variant

This design enables flexible memory management where the heap can expand as needed while maintaining the performance characteristics of the hybrid allocation strategy.