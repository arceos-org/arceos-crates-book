# Architecture Overview

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/lib.rs)
> * [src/linked_list.rs](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/linked_list.rs)
> * [src/raw_list.rs](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs)

This document explains the three-layer architecture of the `linked_list_r4l` crate, detailing how the unsafe raw list implementation, safe wrapper layer, and user-friendly API work together to provide constant-time arbitrary removal with memory safety guarantees.

For specific API usage examples, see [Quick Start Guide](/arceos-org/linked_list_r4l/2-quick-start-guide). For detailed trait and struct documentation, see [API Reference](/arceos-org/linked_list_r4l/4-api-reference). For memory safety and thread safety concepts, see [Core Concepts](/arceos-org/linked_list_r4l/5-core-concepts).

## Three-Layer Architecture

The `linked_list_r4l` crate implements a sophisticated three-layer architecture that separates concerns between performance, safety, and usability:

```mermaid
flowchart TD
subgraph subGraph2["Layer 1: Raw Unsafe Layer"]
    RawListStruct["RawList<G: GetLinks>"]
    GetLinksTrait["GetLinks trait"]
    LinksStruct["Links<T> struct"]
    ListEntryStruct["ListEntry<T> struct"]
    AtomicOps["AtomicBool operations"]
end
subgraph subGraph1["Layer 2: Safe Wrapper Layer"]
    WrappedList["List<G: GetLinksWrapped>"]
    WrapperTrait["Wrapper<T> trait"]
    GetLinksWrappedTrait["GetLinksWrapped trait"]
    BoxImpl["Box<T> implementation"]
    ArcImpl["Arc<T> implementation"]
    RefImpl["&T implementation"]
end
subgraph subGraph0["Layer 3: User-Friendly API"]
    DefNodeMacro["def_node! macro"]
    SimpleList["List<T> type alias"]
    GeneratedNodes["Generated Node Types"]
end

DefNodeMacro --> GeneratedNodes
GeneratedNodes --> GetLinksTrait
GetLinksWrappedTrait --> GetLinksTrait
LinksStruct --> AtomicOps
LinksStruct --> ListEntryStruct
RawListStruct --> GetLinksTrait
RawListStruct --> LinksStruct
SimpleList --> WrappedList
WrappedList --> GetLinksWrappedTrait
WrappedList --> RawListStruct
WrappedList --> WrapperTrait
WrapperTrait --> ArcImpl
WrapperTrait --> BoxImpl
WrapperTrait --> RefImpl
```

**Layer 1 (Raw Unsafe)**: Provides maximum performance through direct pointer manipulation and atomic operations. The `RawList<G: GetLinks>` struct handles all unsafe linked list operations.

**Layer 2 (Safe Wrapper)**: Manages memory ownership and provides safe abstractions over the raw layer. The `List<G: GetLinksWrapped>` struct ensures memory safety through the `Wrapper<T>` trait.

**Layer 3 (User-Friendly)**: Offers convenient APIs and code generation. The `def_node!` macro automatically generates node types that implement required traits.

Sources: [src/lib.rs(L1 - L179)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/lib.rs#L1-L179) [src/linked_list.rs(L1 - L355)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/linked_list.rs#L1-L355) [src/raw_list.rs(L1 - L596)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L1-L596)

## Type System and Trait Architecture

The crate's type system uses traits to enable flexibility while maintaining type safety:

```mermaid
flowchart TD
subgraph subGraph3["Wrapper Implementations"]
    BoxWrapper["impl Wrapper<T> for Box<T>"]
    ArcWrapper["impl Wrapper<T> for Arc<T>"]
    RefWrapper["impl Wrapper<T> for &T"]
end
subgraph subGraph2["Generated Types"]
    MacroGenerated["def_node! generated• inner: T• links: Links<Self>• implements GetLinks"]
end
subgraph subGraph1["Data Structures"]
    Links["Links<T> struct• inserted: AtomicBool• entry: UnsafeCell"]
    ListEntry["ListEntry<T> struct• next: Option<NonNull<T>>• prev: Option<NonNull<T>>"]
    RawList["RawList<G> struct• head: Option<NonNull>"]
    List["List<G> struct• list: RawList<G>"]
end
subgraph subGraph0["Core Traits"]
    GetLinks["GetLinks trait• type EntryType• fn get_links()"]
    GetLinksWrapped["GetLinksWrapped trait• type Wrapped• extends GetLinks"]
    Wrapper["Wrapper<T> trait• fn into_pointer()• fn from_pointer()• fn as_ref()"]
end

ArcWrapper --> Wrapper
BoxWrapper --> Wrapper
GetLinksWrapped --> GetLinks
GetLinksWrapped --> Wrapper
Links --> ListEntry
List --> GetLinksWrapped
List --> RawList
MacroGenerated --> GetLinks
RawList --> GetLinks
RawList --> Links
RefWrapper --> Wrapper
```

The `GetLinks` trait ([src/raw_list.rs(L23 - L29)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L23-L29)) allows any type to participate in linked lists by providing access to embedded `Links<T>`. The `Wrapper<T>` trait ([src/linked_list.rs(L18 - L31)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/linked_list.rs#L18-L31)) abstracts over different ownership models, enabling the same list implementation to work with `Box<T>`, `Arc<T>`, and `&T`.

Sources: [src/raw_list.rs(L23 - L29)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L23-L29) [src/linked_list.rs(L18 - L31)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/linked_list.rs#L18-L31) [src/linked_list.rs(L85 - L121)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/linked_list.rs#L85-L121) [src/lib.rs(L11 - L107)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/lib.rs#L11-L107)

## Memory Management Architecture

The crate implements a controlled boundary between safe and unsafe code to manage memory ownership:

```mermaid
flowchart TD
subgraph subGraph2["Unsafe Pointer Zone"]
    NonNullPtrs["NonNull<T>Raw pointers"]
    RawListOps["RawList<G>Unsafe operations"]
    AtomicInsertion["Links<T>.insertedAtomicBool tracking"]
    PointerArithmetic["ListEntry<T>next/prev pointers"]
end
subgraph subGraph1["Safety Boundary"]
    WrapperLayer["Wrapper<T> traitinto_pointer() / from_pointer()"]
    GetLinksWrappedLayer["GetLinksWrapped traittype Wrapped"]
    ListLayer["List<G> structSafe operations"]
end
subgraph subGraph0["Safe Memory Zone"]
    UserCode["User Code"]
    OwnedBox["Box<Node>Heap ownership"]
    SharedArc["Arc<Node>Reference counting"]
    BorrowedRef["&NodeBorrowed reference"]
end

BorrowedRef --> WrapperLayer
GetLinksWrappedLayer --> ListLayer
ListLayer --> NonNullPtrs
NonNullPtrs --> RawListOps
OwnedBox --> WrapperLayer
RawListOps --> AtomicInsertion
RawListOps --> PointerArithmetic
SharedArc --> WrapperLayer
UserCode --> BorrowedRef
UserCode --> OwnedBox
UserCode --> SharedArc
WrapperLayer --> GetLinksWrappedLayer
```

The safety boundary ensures that:

* Owned objects are converted to raw pointers only when inserted
* Raw pointers are converted back to owned objects when removed
* Atomic tracking prevents double-insertion and use-after-free
* Memory lifetime is managed by the wrapper type

Sources: [src/linked_list.rs(L33 - L83)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/linked_list.rs#L33-L83) [src/raw_list.rs(L35 - L66)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L35-L66) [src/raw_list.rs(L74 - L86)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L74-L86)

## Data Flow and Operation Architecture

Operations flow through the abstraction layers with ownership transformations at each boundary:

```

```

The data flow demonstrates how:

1. The macro generates boilerplate `GetLinks` implementations
2. Ownership transfers happen at wrapper boundaries
3. Atomic operations ensure thread-safe insertion tracking
4. Constant-time removal works through direct pointer manipulation

Sources: [src/lib.rs(L11 - L107)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/lib.rs#L11-L107) [src/linked_list.rs(L153 - L162)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/linked_list.rs#L153-L162) [src/linked_list.rs(L201 - L208)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/linked_list.rs#L201-L208) [src/raw_list.rs(L57 - L65)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L57-L65) [src/raw_list.rs(L199 - L235)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L199-L235)

## Cursor and Iterator Architecture

The crate provides both immutable iteration and mutable cursor-based traversal:

```mermaid
flowchart TD
subgraph subGraph2["Raw Layer Implementation"]
    RawIterator["raw_list::Iterator<G>• cursor_front: Cursor• cursor_back: Cursor"]
    RawCursor["raw_list::Cursor<G>• current() -> &T• move_next/prev()"]
    RawCursorMut["raw_list::CursorMut<G>• current() -> &mut T• remove_current()• peek operations"]
    CommonCursor["CommonCursor<G>• cur: Option<NonNull>• move_next/prev logic"]
end
subgraph subGraph1["Cursor-Based Mutation"]
    CursorMut["List<G>::cursor_front_mut()Returns CursorMut<G>"]
    CursorMutOps["CursorMut operations• current() -> &mut T• remove_current()• peek_next/prev()• move_next()"]
end
subgraph subGraph0["High-Level Iteration"]
    ListIter["List<G>::iter()Returns Iterator<G>"]
    ListIterImpl["impl Iterator for Iterator<G>Delegates to raw_list::Iterator"]
end

CursorMut --> CursorMutOps
CursorMutOps --> RawCursorMut
ListIter --> ListIterImpl
ListIterImpl --> RawIterator
RawCursor --> CommonCursor
RawCursorMut --> CommonCursor
RawIterator --> RawCursor
```

Cursors enable safe mutable access to list elements while maintaining the linked list invariants. The `CursorMut<G>` ([src/linked_list.rs(L238 - L276)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/linked_list.rs#L238-L276)) provides ownership-aware removal that properly converts raw pointers back to owned objects.

Sources: [src/linked_list.rs(L139 - L142)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/linked_list.rs#L139-L142) [src/linked_list.rs(L220 - L276)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/linked_list.rs#L220-L276) [src/raw_list.rs(L104 - L106)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L104-L106) [src/raw_list.rs(L270 - L423)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L270-L423) [src/raw_list.rs(L433 - L464)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L433-L464)