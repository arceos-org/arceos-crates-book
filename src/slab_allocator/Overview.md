# Overview

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-org/slab_allocator/blob/3c13499d/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-org/slab_allocator/blob/3c13499d/.gitignore)
> * [Cargo.toml](https://github.com/arceos-org/slab_allocator/blob/3c13499d/Cargo.toml)
> * [src/lib.rs](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs)
> * [src/slab.rs](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/slab.rs)
> * [src/tests.rs](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/tests.rs)

This document provides a comprehensive overview of the `slab_allocator` crate, a hybrid memory allocation system designed for `no_std` environments. The allocator combines the performance benefits of slab allocation for small, fixed-size blocks with the flexibility of buddy allocation for larger, variable-size requests.

For detailed implementation specifics of individual components, see [Core Architecture](/arceos-org/slab_allocator/3-core-architecture). For practical usage examples and setup instructions, see [Getting Started](/arceos-org/slab_allocator/2-getting-started).

## Purpose and Target Environment

The `slab_allocator` crate addresses the critical need for efficient memory management in resource-constrained environments where the standard library is unavailable. It targets embedded systems, kernel development, and real-time applications that require deterministic allocation performance.

The allocator implements a hybrid strategy that provides O(1) allocation for blocks ≤ 4096 bytes through slab allocation, while falling back to a buddy system allocator for larger requests. This design balances performance predictability with memory utilization efficiency.

**Sources:** [Cargo.toml(L8 - L9)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/Cargo.toml#L8-L9) [src/lib.rs(L1 - L3)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L1-L3)

## System Architecture

The allocator's architecture centers around the `Heap` struct, which orchestrates allocation requests between multiple specialized components based on request size and alignment requirements.

### Core Components Mapping

```mermaid
flowchart TD
subgraph subGraph4["Internal Slab Components"]
    FreeBlockList["FreeBlockList<BLK_SIZE>(src/slab.rs:65)"]
    FreeBlock["FreeBlock(src/slab.rs:112)"]
end
subgraph subGraph3["Large Block Allocation"]
    BuddyAllocator["buddy_allocatorbuddy_system_allocator::Heap<32>(src/lib.rs:48)"]
end
subgraph subGraph2["Slab Allocation System"]
    Slab64["slab_64_bytes: Slab<64>(src/lib.rs:41)"]
    Slab128["slab_128_bytes: Slab<128>(src/lib.rs:42)"]
    Slab256["slab_256_bytes: Slab<256>(src/lib.rs:43)"]
    Slab512["slab_512_bytes: Slab<512>(src/lib.rs:44)"]
    Slab1024["slab_1024_bytes: Slab<1024>(src/lib.rs:45)"]
    Slab2048["slab_2048_bytes: Slab<2048>(src/lib.rs:46)"]
    Slab4096["slab_4096_bytes: Slab<4096>(src/lib.rs:47)"]
end
subgraph subGraph1["Allocation Routing"]
    HeapAllocator["HeapAllocator enum(src/lib.rs:27)"]
    layout_to_allocator["layout_to_allocator()(src/lib.rs:208)"]
end
subgraph subGraph0["Primary Interface"]
    Heap["Heap(src/lib.rs:40)"]
end

FreeBlockList --> FreeBlock
Heap --> HeapAllocator
Heap --> layout_to_allocator
Slab1024 --> FreeBlockList
Slab128 --> FreeBlockList
Slab2048 --> FreeBlockList
Slab256 --> FreeBlockList
Slab4096 --> FreeBlockList
Slab512 --> FreeBlockList
Slab64 --> FreeBlockList
layout_to_allocator --> BuddyAllocator
layout_to_allocator --> Slab1024
layout_to_allocator --> Slab128
layout_to_allocator --> Slab2048
layout_to_allocator --> Slab256
layout_to_allocator --> Slab4096
layout_to_allocator --> Slab512
layout_to_allocator --> Slab64
```

**Sources:** [src/lib.rs(L40 - L49)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L40-L49) [src/lib.rs(L27 - L36)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L27-L36) [src/slab.rs(L4 - L7)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/slab.rs#L4-L7) [src/slab.rs(L65 - L68)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/slab.rs#L65-L68)

### Allocation Decision Flow

The system routes allocation requests through a deterministic decision tree based on the `Layout` parameter's size and alignment requirements.

```mermaid
flowchart TD
Request["allocate(layout: Layout)(src/lib.rs:135)"]
Router["layout_to_allocator(&layout)(src/lib.rs:208-226)"]
SizeCheck["Size > 4096?(src/lib.rs:209)"]
Buddy["HeapAllocator::BuddyAllocator(src/lib.rs:210)"]
AlignCheck["Check size and alignmentconstraints (src/lib.rs:211-225)"]
Slab64Check["≤64 && align≤64?(src/lib.rs:211-212)"]
Slab128Check["≤128 && align≤128?(src/lib.rs:213-214)"]
Slab256Check["≤256 && align≤256?(src/lib.rs:215-216)"]
Slab512Check["≤512 && align≤512?(src/lib.rs:217-218)"]
Slab1024Check["≤1024 && align≤1024?(src/lib.rs:219-220)"]
Slab2048Check["≤2048 && align≤2048?(src/lib.rs:221-222)"]
SlabDefault["HeapAllocator::Slab4096Bytes(src/lib.rs:224)"]
Slab64Enum["HeapAllocator::Slab64Bytes"]
Slab128Enum["HeapAllocator::Slab128Bytes"]
Slab256Enum["HeapAllocator::Slab256Bytes"]
Slab512Enum["HeapAllocator::Slab512Bytes"]
Slab1024Enum["HeapAllocator::Slab1024Bytes"]
Slab2048Enum["HeapAllocator::Slab2048Bytes"]
BuddyImpl["buddy_allocator.alloc()(src/lib.rs:158-162)"]
SlabImpl["slab_X_bytes.allocate()(src/lib.rs:137-157)"]

AlignCheck --> Slab1024Check
AlignCheck --> Slab128Check
AlignCheck --> Slab2048Check
AlignCheck --> Slab256Check
AlignCheck --> Slab512Check
AlignCheck --> Slab64Check
AlignCheck --> SlabDefault
Buddy --> BuddyImpl
Request --> Router
Router --> SizeCheck
SizeCheck --> AlignCheck
SizeCheck --> Buddy
Slab1024Check --> Slab1024Enum
Slab1024Enum --> SlabImpl
Slab128Check --> Slab128Enum
Slab128Enum --> SlabImpl
Slab2048Check --> Slab2048Enum
Slab2048Enum --> SlabImpl
Slab256Check --> Slab256Enum
Slab256Enum --> SlabImpl
Slab512Check --> Slab512Enum
Slab512Enum --> SlabImpl
Slab64Check --> Slab64Enum
Slab64Enum --> SlabImpl
SlabDefault --> SlabImpl
```

**Sources:** [src/lib.rs(L135 - L164)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L135-L164) [src/lib.rs(L208 - L226)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L208-L226)

## Memory Management Strategy

The allocator employs a two-tier memory management approach that optimizes for different allocation patterns:

|Allocation Size|Strategy|Allocator|Time Complexity|Use Case|
| --- | --- | --- | --- | --- |
|≤ 64 bytes|Fixed-size slab|Slab<64>|O(1)|Small objects, metadata|
|65-128 bytes|Fixed-size slab|Slab<128>|O(1)|Small structures|
|129-256 bytes|Fixed-size slab|Slab<256>|O(1)|Medium structures|
|257-512 bytes|Fixed-size slab|Slab<512>|O(1)|Small buffers|
|513-1024 bytes|Fixed-size slab|Slab<1024>|O(1)|Medium buffers|
|1025-2048 bytes|Fixed-size slab|Slab<2048>|O(1)|Large structures|
|2049-4096 bytes|Fixed-size slab|Slab<4096>|O(1)|Page-sized allocations|
|> 4096 bytes|Buddy system|buddy_system_allocator|O(log n)|Large buffers, dynamic data|

**Sources:** [src/lib.rs(L208 - L226)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L208-L226) [src/lib.rs(L41 - L48)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L41-L48)

## Dynamic Growth Mechanism

When a slab exhausts its available blocks, the allocator automatically requests additional memory from the buddy allocator in fixed-size chunks defined by `SET_SIZE` (64 blocks per growth operation).

```mermaid
flowchart TD
SlabEmpty["Slab free_block_list empty(src/slab.rs:42)"]
RequestGrowth["Request SET_SIZE * BLK_SIZEfrom buddy allocator(src/slab.rs:43-47)"]
GrowSlab["grow() creates new FreeBlockList(src/slab.rs:26-33)"]
MergeBlocks["Merge new blocks intoexisting free_block_list(src/slab.rs:30-32)"]
ReturnBlock["Return first available block(src/slab.rs:49)"]

GrowSlab --> MergeBlocks
MergeBlocks --> ReturnBlock
RequestGrowth --> GrowSlab
SlabEmpty --> RequestGrowth
```

**Sources:** [src/slab.rs(L35 - L55)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/slab.rs#L35-L55) [src/slab.rs(L26 - L33)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/slab.rs#L26-L33) [src/lib.rs(L24)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L24-L24)

## Platform Support and Testing

The allocator supports multiple embedded architectures and maintains compatibility through extensive CI/CD testing:

|Target Platform|Purpose|Test Coverage|
| --- | --- | --- |
|x86_64-unknown-linux-gnu|Development/testing|Full unit tests|
|x86_64-unknown-none|Bare metal x86_64|Build verification|
|riscv64gc-unknown-none-elf|RISC-V embedded|Build verification|
|aarch64-unknown-none-softfloat|ARM64 embedded|Build verification|

The test suite includes allocation patterns from simple double-usize allocations to complex multi-size scenarios with varying alignment requirements, ensuring robust behavior across different usage patterns.

**Sources:** [.github/workflows/ci.yml(L12)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/.github/workflows/ci.yml#L12-L12) [.github/workflows/ci.yml(L28 - L30)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/.github/workflows/ci.yml#L28-L30) [src/tests.rs(L39 - L163)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/tests.rs#L39-L163)

## Key Features and Constraints

### Design Features

* **`no_std` compatibility**: No dependency on the standard library
* **Deterministic performance**: O(1) allocation for blocks ≤ 4096 bytes
* **Automatic growth**: Dynamic expansion when slabs become exhausted
* **Memory statistics**: Runtime visibility into allocation patterns
* **Multiple alignment support**: Handles various alignment requirements efficiently

### System Constraints

* **Minimum heap size**: 32KB (`MIN_HEAP_SIZE = 0x8000`)
* **Page alignment requirement**: Heap start address must be 4096-byte aligned
* **Growth granularity**: Memory added in 4096-byte increments
* **Slab threshold**: Fixed 4096-byte boundary between slab and buddy allocation

**Sources:** [src/lib.rs(L25)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L25-L25) [src/lib.rs(L59 - L71)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L59-L71) [src/lib.rs(L95 - L106)&emsp;](https://github.com/arceos-org/slab_allocator/blob/3c13499d/src/lib.rs#L95-L106)