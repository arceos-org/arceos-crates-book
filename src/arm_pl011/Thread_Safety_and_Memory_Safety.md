# Thread Safety and Memory Safety

> **Relevant source files**
> * [src/pl011.rs](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs)

This document covers the thread safety and memory safety guarantees provided by the `arm_pl011` crate, focusing on the `Pl011Uart` implementation and its safe abstractions over hardware register access. For general API usage patterns, see [3.1](/arceos-org/arm_pl011/3.1-pl011uart-methods). For hardware register specifications, see [5](/arceos-org/arm_pl011/5-hardware-reference).

## Purpose and Scope

The `arm_pl011` crate provides memory-safe and thread-safe abstractions for PL011 UART hardware access in `no_std` embedded environments. This page examines the explicit `Send` and `Sync` implementations, memory safety guarantees through `NonNull` and `tock-registers`, and safe usage patterns for concurrent access scenarios.

## Send and Sync Implementations

The `Pl011Uart` struct explicitly implements both `Send` and `Sync` traits through unsafe implementations, enabling safe transfer and sharing between threads.

### Safety Guarantees

```mermaid
flowchart TD
Pl011Uart["Pl011Uart struct"]
Send["unsafe impl Send"]
Sync["unsafe impl Sync"]
NonNull["NonNull<Pl011UartRegs> base"]
SendSafety["Transfer between threads"]
SyncSafety["Shared references across threads"]
MemSafety["Non-null pointer guarantee"]
TockRegs["tock-registers abstraction"]
TypeSafe["Type-safe register access"]
VolatileOps["Volatile read/write operations"]
HardwareExclusive["Hardware access exclusive to instance"]
RegisterAtomic["Register operations are atomic"]

NonNull --> MemSafety
NonNull --> TockRegs
Pl011Uart --> NonNull
Pl011Uart --> Send
Pl011Uart --> Sync
Send --> SendSafety
SendSafety --> HardwareExclusive
Sync --> SyncSafety
SyncSafety --> RegisterAtomic
TockRegs --> TypeSafe
TockRegs --> VolatileOps
```

**Send Safety Justification**: Each `Pl011Uart` instance exclusively owns access to its memory-mapped register region. The hardware controller exists at a unique physical address, making transfer between threads safe as long as no aliasing occurs.

**Sync Safety Justification**: PL011 register operations are atomic at the hardware level. The `tock-registers` abstraction ensures volatile access patterns that are safe for concurrent readers.

Sources: [src/pl011.rs(L46 - L47)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L46-L47)

### Implementation Details

|Trait|Safety Requirement|Implementation Rationale|
| --- | --- | --- |
|Send|Safe to transfer ownership between threads|Hardware register access is location-bound, not thread-bound|
|Sync|Safe to share references across threads|Register operations are atomic; concurrent reads are safe|

The implementations rely on hardware-level atomicity guarantees and the absence of internal mutability that could cause data races.

Sources: [src/pl011.rs(L42 - L47)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L42-L47)

## Memory Safety Guarantees

### NonNull Pointer Management

```mermaid
flowchart TD
Construction["Pl011Uart::new(base: *mut u8)"]
NonNullWrap["NonNull::new(base).unwrap()"]
Cast["cast<Pl011UartRegs>()"]
Storage["NonNull<Pl011UartRegs> base"]
RegAccess["regs() -> &Pl011UartRegs"]
UnsafeDeref["unsafe { self.base.as_ref() }"]
SafeWrapper["Safe register interface"]
PanicOnNull["Panic on null pointer"]
ValidityAssumption["Assumes valid memory mapping"]

Cast --> Storage
Construction --> NonNullWrap
NonNullWrap --> Cast
NonNullWrap --> PanicOnNull
RegAccess --> UnsafeDeref
Storage --> RegAccess
UnsafeDeref --> SafeWrapper
UnsafeDeref --> ValidityAssumption
```

The `Pl011Uart` constructor uses `NonNull::new().unwrap()` to ensure non-null pointer storage, panicking immediately on null input rather than deferring undefined behavior.

**Memory Safety Properties**:

* **Non-null guarantee**: `NonNull<T>` type prevents null pointer dereference
* **Const construction**: Construction in const context validates pointer at compile time when possible
* **Type safety**: Cast to `Pl011UartRegs` provides typed access to register layout

Sources: [src/pl011.rs(L51 - L54)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L51-L54) [src/pl011.rs(L57 - L59)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L57-L59)

### Register Access Safety

The crate uses `tock-registers` to provide memory-safe register access patterns:

```mermaid
flowchart TD
UnsafeDeref["unsafe { self.base.as_ref() }"]
TockInterface["tock-registers interface"]
ReadOnly["ReadOnly<u32> registers"]
ReadWrite["ReadWrite<u32> registers"]
WriteOnly["WriteOnly<u32> registers"]
VolatileRead["Volatile read operations"]
VolatileReadWrite["Volatile read/write operations"]
VolatileWrite["Volatile write operations"]
CompilerBarrier["Prevents compiler reordering"]
HardwareSync["Synchronized with hardware state"]

CompilerBarrier --> HardwareSync
ReadOnly --> VolatileRead
ReadWrite --> VolatileReadWrite
TockInterface --> ReadOnly
TockInterface --> ReadWrite
TockInterface --> WriteOnly
UnsafeDeref --> TockInterface
VolatileRead --> CompilerBarrier
VolatileReadWrite --> CompilerBarrier
VolatileWrite --> CompilerBarrier
WriteOnly --> VolatileWrite
```

**Safety Mechanisms**:

* **Volatile access**: All register operations use volatile semantics to prevent compiler optimizations
* **Type safety**: Register types prevent incorrect access patterns (e.g., writing to read-only registers)
* **Memory barriers**: Volatile operations provide necessary compiler barriers for hardware synchronization

Sources: [src/pl011.rs(L9 - L32)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L9-L32) [src/pl011.rs(L57 - L59)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L57-L59)

## Thread Safety Considerations

### Concurrent Access Patterns

```mermaid
flowchart TD
subgraph subGraph2["Method Safety"]
    ConstMethods["&self methods (read-only)"]
    MutMethods["&mut self methods (exclusive)"]
end
subgraph subGraph1["Unsafe Patterns"]
    AliasedMutable["Multiple mutable references"]
    UnprotectedShared["Unprotected shared mutable access"]
end
subgraph subGraph0["Safe Patterns"]
    ExclusiveOwnership["Exclusive ownership per thread"]
    ReadOnlySharing["Shared read-only access"]
    MutexWrapped["Mutex<Pl011Uart> for shared writes"]
end
SafeTransfer["Safe: move between threads"]
ReadRegisters["Read: fr, mis, ris registers"]
WriteRegisters["Write: dr, cr, imsc, icr registers"]
DataRace["Data race on register access"]
UndefinedBehavior["Undefined behavior"]

AliasedMutable --> DataRace
ConstMethods --> ReadRegisters
ExclusiveOwnership --> SafeTransfer
MutMethods --> WriteRegisters
MutexWrapped --> MutMethods
ReadOnlySharing --> ConstMethods
UnprotectedShared --> UndefinedBehavior
```

**Thread Safety Model**:

* **Shared immutable access**: Multiple threads can safely call `&self` methods for reading status
* **Exclusive mutable access**: Only one thread should have `&mut` access for configuration and I/O operations
* **Hardware atomicity**: Individual register operations are atomic at the hardware level

Sources: [src/pl011.rs(L78 - L103)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L78-L103)

### Method Safety Classification

|Method|Signature|Thread Safety|Hardware Impact|
| --- | --- | --- | --- |
|is_receive_interrupt()|&self|Safe for concurrent access|Read-only status check|
|putchar()|&mut self|Requires exclusive access|Modifies transmit state|
|getchar()|&mut self|Requires exclusive access|Modifies receive state|
|init()|&mut self|Requires exclusive access|Modifies controller configuration|
|ack_interrupts()|&mut self|Requires exclusive access|Clears interrupt state|

The distinction between `&self` and `&mut self` methods reflects the underlying hardware behavior and safety requirements.

Sources: [src/pl011.rs(L64 - L103)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L64-L103)

## Safe Usage Patterns

### Recommended Concurrent Access

```mermaid
flowchart TD
SingleOwner["Single-threaded ownership"]
DirectAccess["Direct method calls"]
MultipleReaders["Multiple reader threads"]
SharedRef["Arc<Pl011Uart>"]
ReadOnlyOps["Only &self methods"]
SharedWriter["Shared writer access"]
MutexWrap["Arc<Mutex<Pl011Uart>>"]
ExclusiveLock["lock().unwrap()"]
FullAccess["All methods available"]
InterruptHandler["Interrupt context"]
AtomicCheck["Atomic status check"]
DeferredWork["Defer work to thread context"]

AtomicCheck --> DeferredWork
ExclusiveLock --> FullAccess
InterruptHandler --> AtomicCheck
MultipleReaders --> SharedRef
MutexWrap --> ExclusiveLock
SharedRef --> ReadOnlyOps
SharedWriter --> MutexWrap
SingleOwner --> DirectAccess
```

**Best Practices**:

1. **Single ownership**: Preferred pattern for most embedded applications
2. **Arc sharing**: Use for status monitoring across multiple threads
3. **Mutex protection**: Required for shared mutable access
4. **Interrupt safety**: Minimize work in interrupt context, use atomic operations only

Sources: [src/pl011.rs(L94 - L96)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L94-L96)

### Initialization Safety

The `const fn new()` constructor enables compile-time safety verification when used with known memory addresses:

```javascript
// Safe: Address known at compile time
const UART0: Pl011Uart = Pl011Uart::new(0x9000_0000 as *mut u8);

// Runtime construction - requires valid address
let uart = Pl011Uart::new(base_addr);
```

**Safety Requirements**:

* Base address must point to valid PL011 hardware registers
* Memory region must remain mapped for lifetime of `Pl011Uart` instance
* No other code should directly access the same register region

Sources: [src/pl011.rs(L50 - L55)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L50-L55)