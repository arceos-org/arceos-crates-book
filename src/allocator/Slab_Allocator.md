# Slab Allocator

> **Relevant source files**
> * [src/slab.rs](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs)

This document covers the `SlabByteAllocator` implementation, which provides byte-granularity memory allocation using the slab allocation algorithm. The slab allocator is designed for efficient allocation and deallocation of fixed-size objects by pre-allocating chunks of memory organized into "slabs."

For information about other byte-level allocators, see [Buddy System Allocator](/arceos-org/allocator/3.2-buddy-system-allocator) and [TLSF Allocator](/arceos-org/allocator/3.4-tlsf-allocator). For page-granularity allocation, see [Bitmap Page Allocator](/arceos-org/allocator/3.1-bitmap-page-allocator). For the overall trait architecture that this allocator implements, see [Architecture and Design](/arceos-org/allocator/2-architecture-and-design).

## Architecture and Design

### Trait Implementation Hierarchy

The `SlabByteAllocator` implements both core allocator traits defined in the crate's trait system:

```mermaid
flowchart TD
BaseAlloc["BaseAllocator"]
ByteAlloc["ByteAllocator"]
SlabImpl["SlabByteAllocator"]
HeapWrapper["slab_allocator::Heap"]
BaseInit["init(start, size)"]
BaseAdd["add_memory(start, size)"]
ByteAllocMethod["alloc(layout)"]
ByteDealloc["dealloc(pos, layout)"]
ByteStats["total_bytes(), used_bytes(), available_bytes()"]
SlabNew["new()"]
SlabInner["inner_mut(), inner()"]

BaseAlloc --> BaseAdd
BaseAlloc --> BaseInit
BaseAlloc --> SlabImpl
ByteAlloc --> ByteAllocMethod
ByteAlloc --> ByteDealloc
ByteAlloc --> ByteStats
ByteAlloc --> SlabImpl
SlabImpl --> HeapWrapper
SlabImpl --> SlabInner
SlabImpl --> SlabNew
```

**Sources:** [src/slab.rs(L5 - L68)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs#L5-L68)

### External Dependency Integration

The `SlabByteAllocator` serves as a wrapper around the external `slab_allocator` crate, specifically the `Heap` type:

```mermaid
flowchart TD
UserCode["User Code"]
SlabWrapper["SlabByteAllocator"]
InternalHeap["Option"]
ExternalCrate["slab_allocator v0.3.1"]
TraitMethods["BaseAllocator + ByteAllocator methods"]
HeapMethods["Heap::new(), allocate(), deallocate()"]

InternalHeap --> ExternalCrate
InternalHeap --> HeapMethods
SlabWrapper --> InternalHeap
SlabWrapper --> TraitMethods
UserCode --> SlabWrapper
```

**Sources:** [src/slab.rs(L8)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs#L8-L8) [src/slab.rs(L14)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs#L14-L14)

## Implementation Details

### Struct Definition and State Management

The `SlabByteAllocator` maintains minimal internal state, delegating the actual allocation logic to the wrapped `Heap`:

|Component|Type|Purpose|
| --- | --- | --- |
|inner|Option<Heap>|Wrapped slab allocator instance|

The `Option` wrapper allows for lazy initialization through the `BaseAllocator::init` method.

**Sources:** [src/slab.rs(L13 - L15)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs#L13-L15)

### Construction and Initialization

```mermaid
sequenceDiagram
    participant User as User
    participant SlabByteAllocator as SlabByteAllocator
    participant Heap as Heap

    User ->> SlabByteAllocator: new()
    Note over Heap,SlabByteAllocator: inner = None
    User ->> SlabByteAllocator: init(start, size)
    SlabByteAllocator ->> Heap: unsafe Heap::new(start, size)
    Heap -->> SlabByteAllocator: heap instance
    Note over Heap,SlabByteAllocator: inner = Some(heap)
```

The allocator follows a two-phase initialization pattern:

1. `new()` creates an uninitialized allocator with `inner = None`
2. `init(start, size)` creates the underlying `Heap` instance

**Sources:** [src/slab.rs(L18 - L21)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs#L18-L21) [src/slab.rs(L33 - L35)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs#L33-L35)

### Memory Management Operations

The allocator provides internal accessor methods for safe access to the initialized heap:

```mermaid
flowchart TD
AccessMethods["Accessor Methods"]
InnerMut["inner_mut() -> &mut Heap"]
Inner["inner() -> &Heap"]
Unwrap["Option::as_mut().unwrap()"]
MutOperations["alloc(), dealloc(), add_memory()"]
ReadOperations["total_bytes(), used_bytes(), available_bytes()"]

AccessMethods --> Inner
AccessMethods --> InnerMut
Inner --> ReadOperations
Inner --> Unwrap
InnerMut --> MutOperations
InnerMut --> Unwrap
```

**Sources:** [src/slab.rs(L23 - L29)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs#L23-L29)

## Memory Allocation API

### BaseAllocator Implementation

The `BaseAllocator` trait provides fundamental memory region management:

|Method|Parameters|Behavior|
| --- | --- | --- |
|init|start: usize, size: usize|Creates newHeapinstance for memory region|
|add_memory|start: usize, size: usize|Adds additional memory region to existing heap|

Both methods work with raw memory addresses and sizes in bytes.

**Sources:** [src/slab.rs(L32 - L43)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs#L32-L43)

### ByteAllocator Implementation

The `ByteAllocator` trait provides fine-grained allocation operations:

```mermaid
flowchart TD
ByteAllocator["ByteAllocator Trait"]
AllocMethod["alloc(layout: Layout)"]
DeallocMethod["dealloc(pos: NonNull, layout: Layout)"]
StatsGroup["Memory Statistics"]
AllocResult["AllocResult>"]
ErrorMap["map_err(|_| AllocError::NoMemory)"]
UnsafeDealloc["unsafe deallocate(ptr, layout)"]
TotalBytes["total_bytes()"]
UsedBytes["used_bytes()"]
AvailableBytes["available_bytes()"]

AllocMethod --> AllocResult
AllocMethod --> ErrorMap
ByteAllocator --> AllocMethod
ByteAllocator --> DeallocMethod
ByteAllocator --> StatsGroup
DeallocMethod --> UnsafeDealloc
StatsGroup --> AvailableBytes
StatsGroup --> TotalBytes
StatsGroup --> UsedBytes
```

### Error Handling

The allocation method converts internal allocation failures to the standardized `AllocError::NoMemory` variant, providing consistent error semantics across all allocator implementations in the crate.

**Sources:** [src/slab.rs(L45 - L68)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs#L45-L68)

### Memory Statistics

The allocator exposes three memory statistics through delegation to the underlying `Heap`:

|Statistic|Description|
| --- | --- |
|total_bytes()|Total memory managed by allocator|
|used_bytes()|Currently allocated memory|
|available_bytes()|Free memory available for allocation|

**Sources:** [src/slab.rs(L57 - L67)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs#L57-L67)

## Slab Allocation Characteristics

The slab allocator is particularly efficient for workloads involving:

* Frequent allocation and deallocation of similar-sized objects
* Scenarios where memory fragmentation needs to be minimized
* Applications requiring predictable allocation performance

The underlying `slab_allocator::Heap` implementation handles the complexity of slab management, chunk organization, and free list maintenance, while the wrapper provides integration with the crate's trait system and error handling conventions.

**Sources:** [src/slab.rs(L1 - L3)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs#L1-L3) [src/slab.rs(L10 - L12)&emsp;](https://github.com/arceos-org/allocator/blob/1d5b7a1b/src/slab.rs#L10-L12)