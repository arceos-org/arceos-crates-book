# SpinRaw

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/kspin/blob/dfc0ff2c/README.md)
> * [src/lib.rs](https://github.com/arceos-org/kspin/blob/dfc0ff2c/src/lib.rs)

## Purpose and Scope

This page documents `SpinRaw<T>`, the raw spinlock implementation in the kspin crate that provides no built-in protection mechanisms. `SpinRaw` offers the fastest performance among the three spinlock types but requires manual management of preemption and interrupt state by the caller.

For information about spinlocks with built-in preemption protection, see [SpinNoPreempt](/arceos-org/kspin/2.2-spinnopreempt). For full protection including IRQ disabling, see [SpinNoIrq](/arceos-org/kspin/2.3-spinnoirq). For detailed implementation internals, see [BaseSpinLock and BaseSpinLockGuard](/arceos-org/kspin/3.1-basespinlock-and-basespinlockguard). For comprehensive usage guidelines across all spinlock types, see [Usage Guidelines and Safety](/arceos-org/kspin/2.4-usage-guidelines-and-safety).

## Type Definition and Basic Interface

`SpinRaw<T>` is implemented as a type alias that specializes the generic `BaseSpinLock` with the `NoOp` guard type from the `kernel_guard` crate.

### Core Type Definitions

```mermaid
flowchart TD
subgraph subGraph0["SpinRaw Type System"]
    SpinRaw["SpinRaw<T>Type alias"]
    BaseSpinLock["BaseSpinLock<NoOp, T>Generic implementation"]
    SpinRawGuard["SpinRawGuard<'a, T>RAII guard"]
    BaseSpinLockGuard["BaseSpinLockGuard<'a, NoOp, T>Generic guard"]
    NoOp["NoOpkernel_guard guard type"]
end

BaseSpinLock --> BaseSpinLockGuard
BaseSpinLock --> NoOp
BaseSpinLockGuard --> NoOp
SpinRaw --> BaseSpinLock
SpinRawGuard --> BaseSpinLockGuard
```

Sources: [src/lib.rs(L33 - L36)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/src/lib.rs#L33-L36)

The `SpinRaw<T>` type provides the standard spinlock interface methods inherited from `BaseSpinLock`:

|Method|Return Type|Description|
| --- | --- | --- |
|new(data: T)|SpinRaw<T>|Creates a new spinlock containing the given data|
|lock()|SpinRawGuard<'_, T>|Acquires the lock, spinning until successful|
|try_lock()|Option<SpinRawGuard<'_, T>>|Attempts to acquire the lock without spinning|

## Protection Behavior and NoOp Guard

The defining characteristic of `SpinRaw` is its use of the `NoOp` guard type, which performs no protection actions during lock acquisition or release.

### Protection Flow Diagram

```mermaid
flowchart TD
subgraph subGraph1["NoOp Guard Behavior"]
    NoOpAcquire["NoOp::acquire()Does nothing"]
    NoOpRelease["NoOp::release()Does nothing"]
end
subgraph subGraph0["SpinRaw Lock Acquisition Flow"]
    Start["lock() called"]
    TryAcquire["Attempt atomic acquisition"]
    IsLocked["Lock alreadyheld?"]
    Spin["Spin wait(busy loop)"]
    Acquired["Lock acquired"]
    GuardCreated["SpinRawGuard createdwith NoOp::acquire()"]
end
note1["Note: No preemption disablingNo IRQ disablingNo protection barriers"]

Acquired --> GuardCreated
GuardCreated --> NoOpAcquire
GuardCreated --> NoOpRelease
IsLocked --> Acquired
IsLocked --> Spin
NoOpAcquire --> note1
NoOpRelease --> note1
Spin --> TryAcquire
Start --> TryAcquire
TryAcquire --> IsLocked
```

Sources: [src/lib.rs(L29 - L36)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/src/lib.rs#L29-L36) [src/base.rs](https://github.com/arceos-org/kspin/blob/dfc0ff2c/src/base.rs)

Unlike `SpinNoPreempt` and `SpinNoIrq`, the `NoOp` guard performs no system-level protection operations:

* No preemption disabling/enabling
* No interrupt disabling/enabling
* No memory barriers beyond those required for the atomic lock operations

## Safety Requirements and Usage Contexts

`SpinRaw` places the burden of ensuring safe usage entirely on the caller. The lock itself provides only the basic mutual exclusion mechanism without any system-level protections.

### Required Caller Responsibilities

```mermaid
flowchart TD
subgraph subGraph1["Consequences of Violation"]
    Deadlock["Potential deadlockif preempted while holding lock"]
    RaceCondition["Race conditionswith interrupt handlers"]
    DataCorruption["Data corruptionfrom concurrent access"]
end
subgraph subGraph0["Caller Must Ensure"]
    PreemptDisabled["Preemption already disabledBefore calling lock()"]
    IRQDisabled["Local IRQs already disabledBefore calling lock()"]
    NoInterruptUse["Never used ininterrupt handlers"]
    ProperContext["Operating in appropriatekernel context"]
end
subgraph subGraph2["Safe Usage Pattern"]
    DisablePreempt["disable_preemption()"]
    DisableIRQ["disable_local_irq()"]
    UseLock["SpinRaw::lock()"]
    CriticalSection["/* critical section */"]
    ReleaseLock["drop(guard)"]
    EnableIRQ["enable_local_irq()"]
    EnablePreempt["enable_preemption()"]
end

CriticalSection --> ReleaseLock
DisableIRQ --> UseLock
DisablePreempt --> DisableIRQ
EnableIRQ --> EnablePreempt
IRQDisabled --> RaceCondition
NoInterruptUse --> RaceCondition
PreemptDisabled --> Deadlock
ProperContext --> DataCorruption
ReleaseLock --> EnableIRQ
UseLock --> CriticalSection
```

Sources: [src/lib.rs(L31 - L32)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/src/lib.rs#L31-L32) [README.md(L19 - L22)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/README.md#L19-L22)

## Performance Characteristics

`SpinRaw` offers the highest performance among the three spinlock types due to its minimal overhead approach.

### Performance Comparison

|Spinlock Type|Acquisition Overhead|Release Overhead|Context Switches|
| --- | --- | --- | --- |
|SpinRaw|Atomic operation only|Atomic operation only|Manual control required|
|SpinNoPreempt|Atomic + preemption disable|Atomic + preemption enable|Prevented during lock|
|SpinNoIrq|Atomic + preemption + IRQ disable|Atomic + preemption + IRQ enable|Fully prevented|

### SMP vs Single-Core Behavior

```mermaid
flowchart TD
subgraph subGraph2["Feature Flag Impact on SpinRaw"]
    SMPEnabled["smp featureenabled?"]
    subgraph subGraph1["Single-Core Path (no smp)"]
        NoLockField["No lock fieldoptimized out"]
        NoAtomics["No atomic operationslock always succeeds"]
        NoSpinning["No spinning behaviorimmediate success"]
        NoOverhead["Zero overheadcompile-time optimization"]
    end
    subgraph subGraph0["Multi-Core Path (smp)"]
        AtomicField["AtomicBool lock fieldin BaseSpinLock"]
        CompareExchange["compare_exchange operationsfor acquisition"]
        RealSpin["Actual spinning behavioron contention"]
        MemoryOrder["Memory orderingconstraints"]
    end
end

AtomicField --> CompareExchange
CompareExchange --> RealSpin
NoAtomics --> NoSpinning
NoLockField --> NoAtomics
NoSpinning --> NoOverhead
RealSpin --> MemoryOrder
SMPEnabled --> AtomicField
SMPEnabled --> NoLockField
```

Sources: [README.md(L12)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/README.md#L12-L12) [src/base.rs](https://github.com/arceos-org/kspin/blob/dfc0ff2c/src/base.rs)

In single-core environments (without the `smp` feature), `SpinRaw` becomes a zero-cost abstraction since no actual locking is necessary when preemption and interrupts are properly controlled.

## Integration with BaseSpinLock Architecture

`SpinRaw` leverages the generic `BaseSpinLock` implementation while providing no additional protection through its `NoOp` guard parameter.

### Architectural Mapping

```mermaid
classDiagram
class SpinRaw {
    <<type alias>>
    
    +new(data: T) SpinRaw~T~
    +lock() SpinRawGuard~T~
    +try_lock() Option~SpinRawGuard~T~~
}

class BaseSpinLock~NoOp_T~ {
    -lock_state: AtomicBool
    +new(data: T) Self
    +lock() BaseSpinLockGuard~NoOp_T~
    +try_lock() Option~BaseSpinLockGuard~NoOp_T~~
    -try_lock_inner() bool
}

class SpinRawGuard {
    <<type alias>>
    
    +deref() &T
    +deref_mut() &mut T
}

class BaseSpinLockGuard~NoOp_T~ {
    -lock: BaseSpinLock~NoOp_T~
    -guard_state: NoOp
    +new() Self
    +deref() &T
    +deref_mut() &mut T
}

class NoOp {
    
    +acquire() Self
    +release()
}

SpinRaw  --|>  BaseSpinLock : type alias
SpinRawGuard  --|>  BaseSpinLockGuard : type alias
BaseSpinLock  -->  BaseSpinLockGuard : creates
BaseSpinLockGuard  -->  NoOp : uses
```

Sources: [src/lib.rs(L33 - L36)&emsp;](https://github.com/arceos-org/kspin/blob/dfc0ff2c/src/lib.rs#L33-L36) [src/base.rs](https://github.com/arceos-org/kspin/blob/dfc0ff2c/src/base.rs)

The `SpinRaw` type inherits all functionality from `BaseSpinLock` while the `NoOp` guard ensures minimal overhead by performing no protection operations during the guard's lifetime.