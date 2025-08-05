# TimerList Data Structure

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs)

This document provides detailed technical documentation of the `TimerList` struct, which serves as the core data structure for managing timed events in the timer_list crate. It covers the internal min-heap architecture, public methods, performance characteristics, and implementation details.

For information about the `TimerEvent` trait and event callback system, see [TimerEvent System](/arceos-org/timer_list/2.2-timerevent-system). For practical usage examples, see [Usage Guide and Examples](/arceos-org/timer_list/3-usage-guide-and-examples).

## Core Structure Overview

The `TimerList<E: TimerEvent>` struct provides an efficient priority queue implementation for managing timed events. It internally uses a binary heap data structure to maintain events in deadline order, enabling O(log n) insertions and O(1) access to the next expiring event.

### Primary Components

```mermaid
flowchart TD
subgraph subGraph0["Type Constraints"]
    TC["E: TimerEvent"]
end
TL["TimerList<E>"]
BH["BinaryHeap<TimerEventWrapper<E>>"]
TEW1["TimerEventWrapper<E>"]
TEW2["TimerEventWrapper<E>"]
TEW3["TimerEventWrapper<E>"]
DL1["deadline: TimeValue"]
EV1["event: E"]
DL2["deadline: TimeValue"]
EV2["event: E"]
DL3["deadline: TimeValue"]
EV3["event: E"]

BH --> TEW1
BH --> TEW2
BH --> TEW3
TEW1 --> DL1
TEW1 --> EV1
TEW2 --> DL2
TEW2 --> EV2
TEW3 --> DL3
TEW3 --> EV3
TL --> BH
TL --> TC
```

**Sources:** [src/lib.rs(L30 - L32)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L30-L32) [src/lib.rs(L21 - L24)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L21-L24)

The structure consists of three main components:

|Component|Type|Purpose|
| --- | --- | --- |
|TimerList<E>|Public struct|Main interface for timer management|
|BinaryHeap<TimerEventWrapper<E>>|Internal field|Heap storage for efficient ordering|
|TimerEventWrapper<E>|Internal struct|Wrapper containing deadline and event data|

## Min-Heap Architecture

The `TimerList` implements a min-heap through custom ordering logic on `TimerEventWrapper<E>`. This ensures that events with earlier deadlines are always at the top of the heap for efficient retrieval.

### Heap Ordering Implementation

```mermaid
flowchart TD
subgraph subGraph2["Trait Implementations"]
    ORD["Ord"]
    PORD["PartialOrd"]
    EQT["Eq"]
    PEQ["PartialEq"]
end
subgraph subGraph1["Min-Heap Property"]
    ROOT["Earliest Deadline"]
    L1["Later Deadline"]
    R1["Later Deadline"]
    L2["Even Later"]
    R2["Even Later"]
end
subgraph subGraph0["TimerEventWrapper Ordering"]
    CMP["cmp()"]
    REV["other.deadline.cmp(&self.deadline)"]
    PCMP["partial_cmp()"]
    EQ["eq()"]
    COMP["self.deadline == other.deadline"]
end

CMP --> REV
CMP --> ROOT
EQ --> COMP
EQT --> EQ
L1 --> L2
L1 --> R2
ORD --> CMP
PCMP --> CMP
PEQ --> EQ
PORD --> PCMP
ROOT --> L1
ROOT --> R1
```

**Sources:** [src/lib.rs(L34 - L52)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L34-L52)

The ordering implementation uses reversed comparison logic:

|Trait|Implementation|Purpose|
| --- | --- | --- |
|Ord::cmp()|other.deadline.cmp(&self.deadline)|Reverses natural ordering for min-heap|
|PartialOrd::partial_cmp()|Delegates tocmp()|Required for heap operations|
|Eq/PartialEq::eq()|self.deadline == other.deadline|Deadline-based equality|

## Public Methods

The `TimerList` provides a clean API for timer management operations:

### Core Operations

|Method|Signature|Complexity|Purpose|
| --- | --- | --- | --- |
|new()|fn new() -> Self|O(1)|Creates empty timer list|
|set()|fn set(&mut self, deadline: TimeValue, event: E)|O(log n)|Schedules new event|
|expire_one()|fn expire_one(&mut self, now: TimeValue) -> Option<(TimeValue, E)>|O(log n)|Processes earliest expired event|
|cancel()|fn cancel<F>(&mut self, condition: F)|O(n)|Removes events matching condition|

### Query Operations

|Method|Signature|Complexity|Purpose|
| --- | --- | --- | --- |
|is_empty()|fn is_empty(&self) -> bool|O(1)|Checks if any events exist|
|next_deadline()|fn next_deadline(&self) -> Option<TimeValue>|O(1)|Returns earliest deadline|

**Sources:** [src/lib.rs(L54 - L100)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L54-L100)

## Internal Implementation Details

### Event Scheduling Flow

```mermaid
sequenceDiagram
    participant Client as Client
    participant TimerList as TimerList
    participant BinaryHeap as BinaryHeap
    participant TimerEventWrapper as TimerEventWrapper

    Client ->> TimerList: "set(deadline, event)"
    TimerList ->> TimerEventWrapper: "TimerEventWrapper { deadline, event }"
    TimerList ->> BinaryHeap: "push(wrapper)"
    Note over BinaryHeap: "Auto-reorders via Ord trait"
    BinaryHeap -->> TimerList: "Heap maintains min property"
    TimerList -->> Client: "Event scheduled"
```

**Sources:** [src/lib.rs(L69 - L71)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L69-L71)

### Event Expiration Flow

```mermaid
sequenceDiagram
    participant Client as Client
    participant TimerList as TimerList
    participant BinaryHeap as BinaryHeap

    Client ->> TimerList: "expire_one(now)"
    TimerList ->> BinaryHeap: "peek()"
    BinaryHeap -->> TimerList: "Some(earliest_wrapper)"
    alt "deadline <= now"
        TimerList ->> BinaryHeap: "pop()"
        BinaryHeap -->> TimerList: "TimerEventWrapper"
        TimerList -->> Client: "Some((deadline, event))"
    else "deadline > now"
        TimerList -->> Client: "None"
    end
```

**Sources:** [src/lib.rs(L92 - L99)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L92-L99)

### Data Structure Memory Layout

The `TimerList` maintains minimal memory overhead with efficient heap allocation:

```mermaid
flowchart TD
subgraph subGraph1["Heap Properties"]
    HP1["Min-heap ordering"]
    HP2["O(log n) rebalancing"]
    HP3["Contiguous memory"]
end
subgraph subGraph0["Memory Layout"]
    TLS["TimerList Struct"]
    BHS["BinaryHeap Storage"]
    VEC["Vec<TimerEventWrapper>"]
    TEW1["TimerEventWrapper { deadline, event }"]
    TEW2["TimerEventWrapper { deadline, event }"]
    TEWN["..."]
end

BHS --> HP1
BHS --> HP2
BHS --> VEC
TLS --> BHS
VEC --> HP3
VEC --> TEW1
VEC --> TEW2
VEC --> TEWN
```

**Sources:** [src/lib.rs(L30 - L32)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L30-L32) [src/lib.rs(L21 - L24)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L21-L24)

## Performance Characteristics

|Operation|Time Complexity|Space Complexity|Notes|
| --- | --- | --- | --- |
|Insert (set)|O(log n)|O(1) additional|Binary heap insertion|
|Peek next (next_deadline)|O(1)|O(1)|Heap root access|
|Pop next (expire_one)|O(log n)|O(1)|Heap rebalancing required|
|Cancel events|O(n)|O(n)|Linear scan with filtering|
|Check empty|O(1)|O(1)|Heap size check|

The min-heap implementation provides optimal performance for the primary use case of sequential event processing by deadline order.

**Sources:** [src/lib.rs(L54 - L100)&emsp;](https://github.com/arceos-org/timer_list/blob/4fa2875f/src/lib.rs#L54-L100)