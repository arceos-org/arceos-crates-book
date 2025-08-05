# Thread Safety & Memory Model

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs)

This document explains the synchronization mechanisms, atomic operations, and memory ordering guarantees that make `LazyInit<T>` thread-safe. It covers the low-level implementation details of how concurrent access is coordinated and memory safety is maintained across multiple threads.

For information about the high-level API and usage patterns, see [API Reference](/arceos-org/lazyinit/2.1-api-reference) and [Usage Patterns & Examples](/arceos-org/lazyinit/2.3-usage-patterns-and-examples).

## Synchronization Primitives

The `LazyInit<T>` implementation relies on two core synchronization primitives to achieve thread safety:

|Component|Type|Purpose|
| --- | --- | --- |
|inited|AtomicBool|Tracks initialization state atomically|
|data|UnsafeCell<MaybeUninit<T>>|Provides interior mutability for the stored value|

### Atomic State Management

The initialization state is tracked using an `AtomicBool` that serves as the primary coordination mechanism between threads. This atomic variable ensures that only one thread can successfully transition from the uninitialized to initialized state.

```mermaid
flowchart TD
A["AtomicBool_inited"]
B["compare_exchange_weak(false, true)"]
C["CAS_Success"]
D["Write_data_initialize"]
E["Already_initialized"]
F["Return_reference"]
G["Handle_according_to_method"]
H["Thread_1"]
I["Thread_2"]
J["Thread_N"]

A --> B
B --> C
C --> D
C --> E
D --> F
E --> G
H --> B
I --> B
J --> B
```

**Atomic State Coordination Diagram**
Sources: [src/lib.rs(L15 - L16)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L15-L16) [src/lib.rs(L37 - L46)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L37-L46) [src/lib.rs(L57 - L66)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L57-L66)

### Interior Mutability Pattern

The `UnsafeCell<MaybeUninit<T>>` provides the necessary interior mutability to allow initialization through shared references while maintaining memory safety through careful synchronization.

```mermaid
flowchart TD
A["UnsafeCell<MaybeUninit<T>>"]
B["get()"]
C["*mut_MaybeUninit<T>"]
D["as_mut_ptr()"]
E["write(data)"]
F["assume_init_ref()"]
G["&T"]
H["Memory_Safety"]
I["Protected_by_AtomicBool"]

A --> B
B --> C
C --> D
C --> F
D --> E
F --> G
H --> I
I --> A
```

**Interior Mutability and Memory Safety**
Sources: [src/lib.rs(L16)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L16-L16) [src/lib.rs(L42)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L42-L42) [src/lib.rs(L119 - L120)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L119-L120)

## Memory Ordering Semantics

The implementation uses specific memory ordering guarantees to ensure correct synchronization across threads while maintaining performance.

### Ordering Strategy

|Operation|Success Ordering|Failure Ordering|Purpose|
| --- | --- | --- | --- |
|compare_exchange_weak|Ordering::Acquire|Ordering::Relaxed|Synchronize initialization|
|loadoperations|Ordering::Acquire|N/A|Ensure visibility of writes|

### Acquire-Release Semantics

```mermaid
sequenceDiagram
    participant Thread_1 as Thread_1
    participant inited_AtomicBool as inited_AtomicBool
    participant data_UnsafeCell as data_UnsafeCell
    participant Thread_2 as Thread_2

    Thread_1 ->> inited_AtomicBool: compare_exchange_weak(false, true, Acquire, Relaxed)
    Note over inited_AtomicBool: CAS succeeds
    inited_AtomicBool -->> Thread_1: Ok(false)
    Thread_1 ->> data_UnsafeCell: write(data)
    Note over Thread_1,data_UnsafeCell: Write becomes visible
    Thread_2 ->> inited_AtomicBool: load(Acquire)
    inited_AtomicBool -->> Thread_2: true
    Note over Thread_2: Acquire guarantees visibility of T1's write
    Thread_2 ->> data_UnsafeCell: assume_init_ref()
    data_UnsafeCell -->> Thread_2: &T (safe)
```

**Memory Ordering Synchronization**
Sources: [src/lib.rs(L38 - L39)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L38-L39) [src/lib.rs(L58 - L59)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L58-L59) [src/lib.rs(L71)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L71-L71)

## Thread Safety Implementation

### Send and Sync Bounds

The trait implementations provide precise thread safety guarantees:

```mermaid
flowchart TD
A["LazyInit<T>"]
B["Send_for_LazyInit<T>"]
C["Sync_for_LazyInit<T>"]
D["T:_Send"]
E["T:Send+_Sync"]
F["Rationale"]
G["Can_transfer_ownership_(Send)"]
H["Can_share_references_(Sync)"]

A --> B
A --> C
B --> D
C --> E
D --> G
E --> H
F --> G
F --> H
```

**Thread Safety Trait Bounds**
Sources: [src/lib.rs(L19 - L20)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L19-L20)

The bounds ensure that:

* `Send` is implemented when `T: Send`, allowing transfer of ownership across threads
* `Sync` is implemented when `T: Send + Sync`, allowing shared access from multiple threads

### Race Condition Prevention

The implementation prevents common race conditions through atomic compare-and-swap operations:

```mermaid
flowchart TD
A["Multiple_threads_call_init_once"]
B["compare_exchange_weak"]
C["First_thread"]
D["Performs_initialization"]
E["Receives_Err"]
F["Writes_data_safely"]
G["Panics_Already_initialized"]
H["call_once_behavior"]
I["Returns_None_for_losers"]
J["No_double_initialization"]
K["Guaranteed_by_CAS"]

A --> B
B --> C
C --> D
C --> E
D --> F
D --> K
E --> G
E --> I
G --> K
H --> I
J --> K
```

**Race Condition Prevention Mechanisms**
Sources: [src/lib.rs(L36 - L46)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L36-L46) [src/lib.rs(L53 - L66)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L53-L66)

## Memory Safety Guarantees

### Initialization State Tracking

The atomic boolean serves as a guard that prevents access to uninitialized memory:

|Method|Check Mechanism|Safety Level|
| --- | --- | --- |
|get()|is_inited()check|Safe - returnsOption<&T>|
|deref()|is_inited()check|Panic on uninitialized|
|get_unchecked()|Debug assertion only|Unsafe - caller responsibility|

### Memory Layout and Access Patterns

```mermaid
flowchart TD
subgraph Unsafe_Access_Path["Unsafe_Access_Path"]
    H["get_unchecked()"]
    I["debug_assert"]
    J["force_get()"]
end
subgraph Safe_Access_Path["Safe_Access_Path"]
    C["is_inited()"]
    D["load(Acquire)"]
    E["true"]
    F["force_get()"]
    G["Return_None_or_Panic"]
end
subgraph LazyInit_Memory_Layout["LazyInit_Memory_Layout"]
    A["inited:_AtomicBool"]
    B["data:_UnsafeCell<MaybeUninit<T>>"]
end
K["assume_init_ref()"]
L["&T"]

A --> C
B --> F
B --> J
C --> D
D --> E
E --> F
E --> G
F --> K
H --> I
I --> J
J --> K
K --> L
```

**Memory Access Safety Mechanisms**
Sources: [src/lib.rs(L77 - L83)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L77-L83) [src/lib.rs(L102 - L105)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L102-L105) [src/lib.rs(L119 - L120)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L119-L120)

## Performance Characteristics

### Fast Path Optimization

Once initialized, access to the value follows a fast path with minimal overhead:

1. **Single atomic load** - `is_inited()` performs one `Ordering::Acquire` load
2. **Direct memory access** - `force_get()` uses `assume_init_ref()` without additional checks
3. **No contention** - Post-initialization reads don't modify atomic state

### Compare-Exchange Optimization

The use of `compare_exchange_weak` instead of `compare_exchange` provides better performance on architectures where weak CAS can fail spuriously but retry efficiently.

```mermaid
flowchart TD
A["compare_exchange_weak"]
B["May_fail_spuriously"]
C["Better_performance_on_some_architectures"]
D["Initialization_path"]
E["One-time_cost"]
F["Amortized_over_lifetime"]
G["Read_path"]
H["Single_atomic_load"]
I["Minimal_overhead"]

A --> B
B --> C
D --> E
E --> F
G --> H
H --> I
```

**Performance Optimization Strategy**
Sources: [src/lib.rs(L38 - L39)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L38-L39) [src/lib.rs(L58 - L59)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L58-L59) [src/lib.rs(L71)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L71-L71)

The memory model ensures that the expensive synchronization cost is paid only once during initialization, while subsequent accesses benefit from the acquire-release semantics without additional synchronization overhead.

Sources: [src/lib.rs(L1 - L183)&emsp;](https://github.com/arceos-org/lazyinit/blob/380d6b07/src/lib.rs#L1-L183)