# Usage Examples

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md)

This document provides practical examples demonstrating how to use the `HandlerTable` API in real-world scenarios. It covers common usage patterns, integration strategies, and best practices for implementing lock-free event handling in no_std environments.

For detailed API documentation, see [API Reference](/arceos-org/handler_table/2.1-api-reference). For implementation details about the underlying atomic operations, see [Implementation Details](/arceos-org/handler_table/3-implementation-details).

## Basic Event Handler Registration

The fundamental usage pattern involves creating a static `HandlerTable`, registering event handlers, and dispatching events as they occur.

### Static Table Creation and Handler Registration

```

```

The basic pattern demonstrated in [README.md(L11 - L31)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L11-L31) shows:

|Operation|Method|Purpose|
| --- | --- | --- |
|Creation|HandlerTable::new()|Initialize empty handler table|
|Registration|register_handler(idx, fn)|Associate handler function with event ID|
|Dispatch|handle(idx)|Execute registered handler for event|
|Cleanup|unregister_handler(idx)|Remove and retrieve handler function|

Sources: [README.md(L11 - L31)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L11-L31)

## Event Registration Patterns

### Single Event Handler Registration

The most common pattern involves registering individual handlers for specific event types. Each handler is a closure or function pointer that gets executed when the corresponding event occurs.

```mermaid
sequenceDiagram
    participant EventSource as "Event Source"
    participant HandlerTableN as "HandlerTable<N>"
    participant AtomicUsizeidx as "AtomicUsize[idx]"
    participant EventHandlerFn as "Event Handler Fn"

    EventSource ->> HandlerTableN: "register_handler(0, closure)"
    HandlerTableN ->> AtomicUsizeidx: "compare_exchange(0, fn_ptr)"
    alt "Registration Success"
        AtomicUsizeidx -->> HandlerTableN: "Ok(0)"
        HandlerTableN -->> EventSource: "true"
    else "Slot Occupied"
        AtomicUsizeidx -->> HandlerTableN: "Err(existing_ptr)"
        HandlerTableN -->> EventSource: "false"
    end
    Note over EventSource,EventHandlerFn: "Later: Event Dispatch"
    EventSource ->> HandlerTableN: "handle(0)"
    HandlerTableN ->> AtomicUsizeidx: "load(Ordering::Acquire)"
    AtomicUsizeidx -->> HandlerTableN: "fn_ptr"
    HandlerTableN ->> EventHandlerFn: "transmute + call()"
    EventHandlerFn -->> HandlerTableN: "execution complete"
    HandlerTableN -->> EventSource: "true"
```

Sources: [README.md(L16 - L21)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L16-L21)

### Multiple Handler Registration

For systems handling multiple event types, handlers are typically registered during initialization:

```mermaid
flowchart TD
subgraph subGraph2["Handler Functions"]
    TimerFn["timer_interrupt_handler"]
    IOFn["io_completion_handler"]
    UserFn["user_signal_handler"]
end
subgraph subGraph1["HandlerTable Operations"]
    Reg0["register_handler(0, timer_fn)"]
    Reg1["register_handler(1, io_fn)"]
    Reg2["register_handler(2, user_fn)"]
end
subgraph subGraph0["System Initialization Phase"]
    Init["System Init"]
    Timer["Timer Handler Setup"]
    IO["I/O Handler Setup"]
    User["User Event Setup"]
end

IO --> Reg1
Init --> IO
Init --> Timer
Init --> User
Reg0 --> TimerFn
Reg1 --> IOFn
Reg2 --> UserFn
Timer --> Reg0
User --> Reg2
```

Sources: [README.md(L16 - L21)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L16-L21)

## Event Handling Scenarios

### Successful Event Dispatch

When an event occurs and a handler is registered, the `handle` method executes the associated function and returns `true`:

```mermaid
flowchart TD
subgraph subGraph0["Example from README"]
    Ex1["handle(0) -> true"]
    Ex2["handle(2) -> false"]
end
Event["Event Occurs"]
Handle["handle(event_id)"]
Check["Check AtomicUsize[event_id]"]
Found["Handler Found?"]
Execute["Execute Handler Function"]
ReturnFalse["Return false"]
ReturnTrue["Return true"]

Check --> Found
Event --> Handle
Execute --> ReturnTrue
Found --> Execute
Found --> ReturnFalse
Handle --> Check
```

The example in [README.md(L23 - L24)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L23-L24) demonstrates both successful dispatch (`handle(0)` returns `true`) and missing handler scenarios (`handle(2)` returns `false`).

Sources: [README.md(L23 - L24)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L23-L24)

### Handler Retrieval and Cleanup

The `unregister_handler` method provides atomic removal and retrieval of handler functions:

```mermaid
flowchart TD
subgraph subGraph2["Example Calls"]
    Unreg1["unregister_handler(2) -> None"]
    Unreg2["unregister_handler(1) -> Some(fn)"]
end
subgraph subGraph1["Return Values"]
    Some["Some(handler_fn)"]
    None["None"]
end
subgraph subGraph0["Unregistration Process"]
    Call["unregister_handler(idx)"]
    Swap["atomic_swap(0)"]
    Check["Check Previous Value"]
    Result["Option"]
end

Call --> Swap
Check --> Result
None --> Unreg1
Result --> None
Result --> Some
Some --> Unreg2
Swap --> Check
```

The pattern shown in [README.md(L26 - L28)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L26-L28) demonstrates retrieving a previously registered handler function, which can then be called directly.

Sources: [README.md(L26 - L30)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L26-L30)

## Advanced Usage Patterns

### Concurrent Handler Management

In multi-threaded environments, multiple threads can safely register, handle, and unregister events without synchronization primitives:

```mermaid
flowchart TD
subgraph subGraph2["Memory Ordering"]
    Acquire["Acquire Ordering"]
    Release["Release Ordering"]
    Consistency["Memory Consistency"]
end
subgraph subGraph1["Atomic Operations Layer"]
    CAS["compare_exchange"]
    Load["load(Acquire)"]
    Swap["swap(Acquire)"]
end
subgraph subGraph0["Thread Safety Guarantees"]
    T1["Thread 1: register_handler"]
    T2["Thread 2: handle"]
    T3["Thread 3: unregister_handler"]
end

Acquire --> Consistency
CAS --> Release
Load --> Acquire
Release --> Consistency
Swap --> Acquire
T1 --> CAS
T2 --> Load
T3 --> Swap
```

Sources: [README.md(L14)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L14-L14)

### Static vs Dynamic Handler Tables

The compile-time sized `HandlerTable<N>` enables static allocation suitable for no_std environments:

|Approach|Declaration|Use Case|
| --- | --- | --- |
|Static Global|static TABLE: HandlerTable<8>|System-wide event handling|
|Local Instance|let table = HandlerTable::<16>::new()|Component-specific events|
|Const Generic|HandlerTable<MAX_EVENTS>|Configurable event capacity|

The static approach shown in [README.md(L14)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L14-L14) is preferred for kernel-level integration where heap allocation is unavailable.

Sources: [README.md(L14)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L14-L14)

## Integration Examples

### ArceOS Kernel Integration

In the ArceOS operating system context, `HandlerTable` serves as the central event dispatch mechanism:

```mermaid
flowchart TD
subgraph subGraph2["Event Dispatch Flow"]
    Dispatch["event_dispatch(id)"]
    Handle["TABLE.handle(id)"]
    Execute["handler_function()"]
end
subgraph subGraph1["HandlerTable Integration"]
    TABLE["static KERNEL_EVENTS: HandlerTable<32>"]
    IRQ["IRQ_HANDLERS[0..15]"]
    TIMER["TIMER_EVENTS[16..23]"]
    SCHED["SCHED_EVENTS[24..31]"]
end
subgraph subGraph0["ArceOS Kernel Components"]
    Interrupt["Interrupt Controller"]
    Timer["Timer Subsystem"]
    Scheduler["Task Scheduler"]
    Syscall["System Call Handler"]
end

Dispatch --> Handle
Handle --> Execute
IRQ --> TABLE
Interrupt --> IRQ
SCHED --> TABLE
Scheduler --> SCHED
Syscall --> TABLE
TABLE --> Dispatch
TIMER --> TABLE
Timer --> TIMER
```

Sources: [README.md(L7)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L7-L7)

### Device Driver Event Handling

Device drivers can use dedicated handler tables for managing I/O completion events:

```mermaid
flowchart TD
subgraph subGraph2["Event Types"]
    Complete["DMA_COMPLETE"]
    Error["DMA_ERROR"]
    Timeout["DMA_TIMEOUT"]
end
subgraph subGraph1["Event Management"]
    DriverTable["HandlerTable"]
    CompletionFn["io_completion_handler"]
    ErrorFn["io_error_handler"]
end
subgraph subGraph0["Driver Architecture"]
    Driver["Device Driver"]
    DMA["DMA Controller"]
    Buffer["I/O Buffers"]
end

Complete --> CompletionFn
CompletionFn --> Buffer
DMA --> Complete
DMA --> Error
DMA --> Timeout
Driver --> DriverTable
Error --> ErrorFn
ErrorFn --> Buffer
Timeout --> ErrorFn
```

This pattern allows drivers to maintain separate event handling contexts while leveraging the same lock-free infrastructure.

Sources: [README.md(L7)&emsp;](https://github.com/arceos-org/handler_table/blob/036a12c4/README.md#L7-L7)