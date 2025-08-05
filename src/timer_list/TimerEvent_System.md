# TimerEvent System

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs)

This document covers the timer event system that defines how callbacks are executed when timers expire in the timer_list crate. It focuses on the `TimerEvent` trait and its implementations, particularly the `TimerEventFn` wrapper for closure-based events.

For information about the underlying timer storage and scheduling mechanisms, see [TimerList Data Structure](/arceos-org/timer_list/2.1-timerlist-data-structure).

## Purpose and Scope

The TimerEvent system provides the interface and implementations for executable timer callbacks in the timer_list crate. It defines how events are structured and executed when their deadlines are reached, supporting both custom event types and simple closure-based events.

## TimerEvent Trait

The `TimerEvent` trait serves as the core interface that all timer events must implement. It defines a single callback method that consumes the event when executed.

### Trait Definition

The trait is defined with a simple interface at [src/lib.rs(L15 - L19)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L15-L19):

```rust
pub trait TimerEvent {
    fn callback(self, now: TimeValue);
}
```

### Key Characteristics

|Aspect|Description|
| --- | --- |
|Consumption|Thecallbackmethod takesselfby value, consuming the event|
|Time Parameter|Receives the current time asTimeValue(alias forDuration)|
|Return Type|Returns()- events perform side effects rather than return values|
|Execution Model|Called exactly once when the timer expires|

The trait design ensures that events cannot be accidentally reused after execution, providing memory safety and preventing common timer-related bugs.

**TimerEvent Trait Architecture**

```mermaid
classDiagram
class TimerEvent {
    <<trait>>
    
    +callback(self, now: TimeValue)
}

class TimerEventFn {
    
    -Box~dyn FnOnce(TimeValue) ~
    +new(f: F) TimerEventFn
    +callback(self, now: TimeValue)
}

class CustomEvent {
    <<user-defined>>
    
    +callback(self, now: TimeValue)
}

TimerEvent  ..|>  TimerEventFn : implements
TimerEvent  ..|>  CustomEvent : implements
```

Sources: [src/lib.rs(L15 - L19)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L15-L19) [src/lib.rs(L108 - L127)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L108-L127)

## TimerEventFn Implementation

The `TimerEventFn` struct provides a convenient wrapper that allows closures to be used as timer events without requiring custom trait implementations.

### Structure and Construction

The `TimerEventFn` wraps a boxed closure at [src/lib.rs(L111)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L111-L111):

```
pub struct TimerEventFn(Box<dyn FnOnce(TimeValue) + 'static>);
```

Construction is handled by the `new` method at [src/lib.rs(L114 - L121)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L114-L121):

|Parameter|Type|Description|
| --- | --- | --- |
|f|F: FnOnce(TimeValue) + 'static|Closure to execute when timer expires|
|Return|TimerEventFn|Wrapped timer event ready for scheduling|

### TimerEvent Implementation

The trait implementation at [src/lib.rs(L123 - L127)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L123-L127) simply extracts and calls the wrapped closure:

```rust
impl TimerEvent for TimerEventFn {
    fn callback(self, now: TimeValue) {
        (self.0)(now)
    }
}
```

This design enables functional programming patterns while maintaining the trait's consumption semantics.

**TimerEventFn Data Flow**

```mermaid
flowchart TD
A["closure: FnOnce(TimeValue)"]
B["TimerEventFn::new(closure)"]
C["TimerEventFn(Box::new(closure))"]
D["TimerList::set(deadline, event)"]
E["TimerEventWrapper { deadline, event }"]
F["BinaryHeap::push(wrapper)"]
G["TimerList::expire_one(now)"]
H["BinaryHeap::pop()"]
I["event.callback(now)"]
J["(closure)(now)"]

A --> B
B --> C
C --> D
D --> E
E --> F
G --> H
H --> I
I --> J
```

Sources: [src/lib.rs(L108 - L127)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L108-L127) [src/lib.rs(L69 - L71)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L69-L71)

## Event Lifecycle Integration

Timer events integrate with the `TimerList` through a well-defined lifecycle that ensures proper execution timing and resource management.

### Event Scheduling Process

Events are scheduled through the `TimerList::set` method, which wraps them in `TimerEventWrapper` structures for heap management:

|Step|Component|Action|
| --- | --- | --- |
|1|Client Code|Creates event implementingTimerEvent|
|2|TimerList::set|Wraps event inTimerEventWrapperwith deadline|
|3|BinaryHeap|Stores wrapper using min-heap ordering|
|4|TimerList::expire_one|Checks earliest deadline against current time|
|5|Event Callback|Executesevent.callback(now)if expired|

### Execution Semantics

When events expire, they follow a strict execution pattern defined at [src/lib.rs(L92 - L99)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L92-L99):

* Events are processed one at a time in deadline order
* Only events with `deadline <= now` are eligible for execution
* Events are removed from the heap before callback execution
* Callbacks receive the actual current time, not the original deadline

**Event Execution Flow**

```mermaid
sequenceDiagram
    participant ClientCode as "Client Code"
    participant TimerList as "TimerList"
    participant BinaryHeapTimerEventWrapper as "BinaryHeap<TimerEventWrapper>"
    participant TimerEventcallback as "TimerEvent::callback"

    Note over ClientCode,TimerEventcallback: Event Scheduling
    ClientCode ->> TimerList: set(deadline, event)
    TimerList ->> BinaryHeapTimerEventWrapper: push(TimerEventWrapper)
    Note over ClientCode,TimerEventcallback: Event Processing
    loop Processing Loop
        ClientCode ->> TimerList: expire_one(now)
        TimerList ->> BinaryHeapTimerEventWrapper: peek()
    alt deadline <= now
        BinaryHeapTimerEventWrapper -->> TimerList: earliest event
        TimerList ->> BinaryHeapTimerEventWrapper: pop()
        TimerList ->> TimerEventcallback: callback(now)
        TimerEventcallback -->> ClientCode: side effects
        TimerList -->> ClientCode: Some((deadline, event))
    else deadline > now
        TimerList -->> ClientCode: None
    end
    end
```

Sources: [src/lib.rs(L69 - L71)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L69-L71) [src/lib.rs(L92 - L99)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L92-L99) [src/lib.rs(L21 - L24)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L21-L24)

## Custom Event Implementation

While `TimerEventFn` handles simple closure cases, custom event types can implement `TimerEvent` directly for more complex scenarios. The test suite demonstrates this pattern at [src/lib.rs(L140 - L153)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L140-L153):

### Example Pattern

```rust
struct TestTimerEvent(usize, TimeValue);

impl TimerEvent for TestTimerEvent {
    fn callback(self, now: TimeValue) {
        // Custom logic with access to event data
        println!("Event {} executed at {:?}", self.0, now);
    }
}
```

This approach allows events to carry additional data and implement complex callback logic while maintaining the same execution guarantees as `TimerEventFn`.

**Code Entity Mapping**

```mermaid
flowchart TD
subgraph subGraph2["Test Examples"]
    H["TestTimerEvent[src/lib.rs:140-153]"]
    I["test_timer_list_fn[src/lib.rs:183-206]"]
end
subgraph subGraph1["Integration Points"]
    E["TimerList::set()[src/lib.rs:69-71]"]
    F["TimerList::expire_one()[src/lib.rs:92-99]"]
    G["TimerEventWrapper[src/lib.rs:21-24]"]
end
subgraph subGraph0["TimerEvent System API"]
    A["TimerEvent trait[src/lib.rs:15-19]"]
    B["TimerEventFn struct[src/lib.rs:111]"]
    C["TimerEventFn::new()[src/lib.rs:114-121]"]
    D["TimeValue type[src/lib.rs:10-13]"]
end

A --> B
A --> D
A --> H
B --> C
B --> I
E --> G
F --> A
```

Sources: [src/lib.rs(L15 - L19)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L15-L19) [src/lib.rs(L108 - L127)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L108-L127) [src/lib.rs(L69 - L71)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L69-L71) [src/lib.rs(L92 - L99)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L92-L99) [src/lib.rs(L140 - L153)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L140-L153) [src/lib.rs(L183 - L206)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L183-L206)