# Atomic Operations

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs)

This document explains the atomic operations that form the foundation of the `HandlerTable`'s lock-free design. It covers the specific atomic primitives used, memory ordering guarantees, and how these operations ensure thread safety without traditional locking mechanisms.

For information about the overall implementation architecture and memory safety aspects, see [Memory Layout and Safety](/arceos-org/handler_table/3.2-memory-layout-and-safety). For practical usage of the lock-free API, see [API Reference](/arceos-org/handler_table/2.1-api-reference).

## Atomic Data Structure Foundation

The `HandlerTable` uses an array of `AtomicUsize` values as its core storage mechanism. Each slot in the array can atomically hold either a null value (0) or a function pointer cast to `usize`.

### Core Atomic Array Structure

```mermaid
flowchart TD
subgraph States["Slot States"]
    Empty["Empty: 0"]
    Occupied["Occupied: handler as usize"]
end
subgraph AtomicSlots["Atomic Storage Slots"]
    Slot0["AtomicUsize[0]Value: 0 or fn_ptr"]
    Slot1["AtomicUsize[1]Value: 0 or fn_ptr"]
    SlotN["AtomicUsize[N-1]Value: 0 or fn_ptr"]
end
subgraph HandlerTable["HandlerTable<N>"]
    Array["handlers: [AtomicUsize; N]"]
end

Array --> Slot0
Array --> Slot1
Array --> SlotN
Slot0 --> Empty
Slot0 --> Occupied
Slot1 --> Empty
Slot1 --> Occupied
SlotN --> Empty
SlotN --> Occupied
```

Sources: [src/lib.rs(L14 - L16)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L14-L16) [src/lib.rs(L21 - L23)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L21-L23)

The atomic array is initialized with zero values using const generics, ensuring compile-time allocation without heap usage. Each `AtomicUsize` slot represents either an empty handler position (value 0) or contains a function pointer cast to `usize`.

## Core Atomic Operations

The `HandlerTable` implements three primary atomic operations that provide lock-free access to the handler storage.

### Compare-Exchange Operation

The `compare_exchange` operation is used in `register_handler` to atomically install a new handler only if the slot is currently empty.

```mermaid
sequenceDiagram
    participant ClientThread as "Client Thread"
    participant AtomicUsizeidx as "AtomicUsize[idx]"
    participant MemorySystem as "Memory System"

    ClientThread ->> AtomicUsizeidx: "compare_exchange(0, handler_ptr, Acquire, Relaxed)"
    AtomicUsizeidx ->> MemorySystem: "Check current value == 0"
    alt Current value is 0
        MemorySystem ->> AtomicUsizeidx: "Replace with handler_ptr"
        AtomicUsizeidx ->> ClientThread: "Ok(0)"
        Note over ClientThread: "Registration successful"
    else Current value != 0
        MemorySystem ->> AtomicUsizeidx: "Keep current value"
        AtomicUsizeidx ->> ClientThread: "Err(current_value)"
        Note over ClientThread: "Registration failed - slot occupied"
    end
```

Sources: [src/lib.rs(L34 - L36)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L34-L36)

The operation uses `Ordering::Acquire` for success and `Ordering::Relaxed` for failure, ensuring proper synchronization when a handler is successfully installed.

### Atomic Swap Operation

The `swap` operation in `unregister_handler` atomically replaces the current handler with zero and returns the previous value.

```mermaid
flowchart TD
Start["swap(0, Ordering::Acquire)"]
Read["Atomically read current value"]
Write["Atomically write 0"]
Check["Current value != 0?"]
Convert["transmute<usize, fn()>(value)"]
ReturnSome["Return Some(handler)"]
ReturnNone["Return None"]

Check --> Convert
Check --> ReturnNone
Convert --> ReturnSome
Read --> Write
Start --> Read
Write --> Check
```

Sources: [src/lib.rs(L46)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L46-L46) [src/lib.rs(L47 - L51)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L47-L51)

### Atomic Load Operation

The `load` operation in `handle` reads the current handler value without modification, using acquire ordering to ensure proper synchronization.

```mermaid
flowchart TD
subgraph Results["Possible Results"]
    Success["Return true"]
    NoHandler["Return false"]
end
subgraph LoadOperation["Atomic Load Process"]
    Load["load(Ordering::Acquire)"]
    Check["Check value != 0"]
    Transmute["unsafe transmute<usize, Handler>"]
    Call["handler()"]
end

Call --> Success
Check --> NoHandler
Check --> Transmute
Load --> Check
Transmute --> Call
```

Sources: [src/lib.rs(L62)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L62-L62) [src/lib.rs(L63 - L66)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L63-L66)

## Memory Ordering Guarantees

The atomic operations use specific memory ordering to ensure correct synchronization across threads while minimizing performance overhead.

### Ordering Usage Patterns

|Operation|Success Ordering|Failure Ordering|Purpose|
| --- | --- | --- | --- |
|compare_exchange|Acquire|Relaxed|Synchronize on successful registration|
|swap|Acquire|N/A|Synchronize when removing handler|
|load|Acquire|N/A|Synchronize when reading handler|

Sources: [src/lib.rs(L35)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L35-L35) [src/lib.rs(L46)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L46-L46) [src/lib.rs(L62)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L62-L62)

### Acquire Ordering Semantics

```mermaid
flowchart TD
subgraph Ordering["Memory Ordering Guarantee"]
    Sync["Acquire ensures visibilityof prior writes"]
end
subgraph Thread2["Thread 2 - Handle Event"]
    T2_Load["load(Acquire)"]
    T2_Read["Read handler data"]
    T2_Call["Call handler"]
end
subgraph Thread1["Thread 1 - Register Handler"]
    T1_Write["Write handler data"]
    T1_CAS["compare_exchange(..., Acquire, ...)"]
end

T1_CAS --> Sync
T1_CAS --> T2_Load
T1_Write --> T1_CAS
T2_Load --> Sync
T2_Load --> T2_Read
T2_Read --> T2_Call
```

Sources: [src/lib.rs(L35)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L35-L35) [src/lib.rs(L62)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L62-L62)

The `Acquire` ordering ensures that when a thread successfully reads a non-zero handler value, it observes all memory writes that happened-before the handler was stored.

## Lock-Free Properties

The atomic operations provide several lock-free guarantees essential for kernel-level event handling.

### Wait-Free Registration

```mermaid
stateDiagram-v2
[*] --> Attempting : "register_handler(idx, fn)"
Attempting --> Success : "CAS succeeds (slot was 0)"
Attempting --> Failure : "CAS fails (slot occupied)"
Success --> [*] : "Return true"
Failure --> [*] : "Return false"
note left of Success : ['"Single atomic operation"']
note left of Failure : ['"No retry loops"']
```

Sources: [src/lib.rs(L30 - L37)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L30-L37)

The registration operation is wait-free - it completes in a bounded number of steps regardless of other thread activity. Either the compare-exchange succeeds immediately, or it fails immediately if another handler is already registered.

### Non-Blocking Event Handling

The `handle` operation never blocks and provides consistent performance characteristics:

* **Constant Time**: Single atomic load operation
* **No Contention**: Multiple threads can simultaneously handle different events
* **Real-Time Safe**: No unbounded waiting or priority inversion

Sources: [src/lib.rs(L58 - L70)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L58-L70)

## Function Pointer Storage Mechanism

The atomic operations work on `usize` values that represent function pointers, requiring careful handling to maintain type safety.

### Pointer-to-Integer Conversion

```mermaid
flowchart TD
subgraph Output["Output Types"]
    Recovered["Handler (fn())"]
end
subgraph Retrieval["Type Recovery"]
    Load["AtomicUsize::load"]
    Transmute["unsafe transmute<usize, fn()>"]
end
subgraph Storage["Atomic Storage"]
    Atomic["AtomicUsize value"]
end
subgraph Conversion["Type Conversion"]
    Cast["handler as usize"]
    Store["AtomicUsize::store"]
end
subgraph Input["Input Types"]
    Handler["Handler (fn())"]
end

Atomic --> Load
Cast --> Store
Handler --> Cast
Load --> Transmute
Store --> Atomic
Transmute --> Recovered
```

Sources: [src/lib.rs(L35)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L35-L35) [src/lib.rs(L48)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L48-L48) [src/lib.rs(L64)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/src/lib.rs#L64-L64)

The conversion relies on the platform guarantee that function pointers can be safely cast to `usize` and back. The `unsafe` transmute operations are necessary because the atomic types only work with integer values, but the conversions preserve the original function pointer values exactly.