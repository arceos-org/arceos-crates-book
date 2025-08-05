# Low-Level API

> **Relevant source files**
> * [src/raw_list.rs](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs)

This page documents the unsafe, low-level API provided by the `RawList<G: GetLinks>` interface for advanced users who need direct control over memory management and performance-critical operations. This layer operates with raw pointers and requires careful attention to memory safety invariants.

For safe ownership management and wrapper abstractions, see [Advanced API](/arceos-org/linked_list_r4l/4.2-advanced-api). For the user-friendly macro-generated interface, see [User-Friendly API](/arceos-org/linked_list_r4l/4.1-user-friendly-api).

## Core Types and Traits

The low-level API is built around three primary types that work together to provide intrusive linked list functionality.

### GetLinks Trait

The `GetLinks` trait serves as the fundamental abstraction that allows any type to be stored in a linked list by providing access to embedded link fields.

```mermaid
flowchart TD
subgraph subGraph1["Implementation Requirements"]
    UserStruct["User Struct(contains Links field)"]
    LinksField["links: Links"]
    ImplBlock["impl GetLinksfor UserStruct"]
end
subgraph subGraph0["GetLinks Trait System"]
    GetLinksTrait["GetLinkstrait"]
    EntryType["type EntryType: ?Sized"]
    GetLinksMethod["get_links(&EntryType)-> &Links"]
end

GetLinksMethod --> LinksField
GetLinksTrait --> EntryType
GetLinksTrait --> GetLinksMethod
ImplBlock --> GetLinksTrait
UserStruct --> ImplBlock
UserStruct --> LinksField
```

**Core GetLinks Implementation Pattern**

The trait defines two key components: an associated type `EntryType` that represents the type of objects stored in the list, and a method `get_links()` that returns a reference to the embedded `Links<T>` structure.

Sources: [src/raw_list.rs(L23 - L29)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L23-L29)

### Links Structure

The `Links<T>` structure contains the actual linked list pointers and provides atomic insertion tracking to prevent double-insertion and enable constant-time removal.

```mermaid
flowchart TD
subgraph subGraph2["Atomic Operations"]
    AcquireInsert["acquire_for_insertion()compare_exchange(false, true)"]
    ReleaseRemove["release_after_removal()store(false, Release)"]
end
subgraph subGraph1["ListEntry Contents"]
    NextPtr["next: Option>"]
    PrevPtr["prev: Option>"]
end
subgraph subGraph0["Links Memory Layout"]
    LinksStruct["Links"]
    InsertedFlag["inserted: AtomicBool(tracks insertion state)"]
    EntryCell["entry: UnsafeCell>(contains pointers)"]
end

EntryCell --> NextPtr
EntryCell --> PrevPtr
InsertedFlag --> AcquireInsert
InsertedFlag --> ReleaseRemove
LinksStruct --> EntryCell
LinksStruct --> InsertedFlag
```

**Atomic Insertion Tracking**

The `inserted` field uses atomic compare-and-swap operations to ensure thread-safe insertion tracking. The `acquire_for_insertion()` method attempts to atomically change the state from `false` to `true`, preventing double-insertion.

Sources: [src/raw_list.rs(L35 - L66)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L35-L66) [src/raw_list.rs(L74 - L86)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L74-L86)

### RawList Structure

The `RawList<G: GetLinks>` provides the core linked list operations using raw pointer manipulation.

```mermaid
flowchart TD
subgraph subGraph3["Iteration Support"]
    Iter["iter() -> Iterator"]
    CursorFront["cursor_front_mut()-> CursorMut"]
end
subgraph subGraph2["Query Operations"]
    IsEmpty["is_empty() -> bool"]
    Front["front() -> Option>"]
    Back["back() -> Option>"]
end
subgraph subGraph1["Primary Operations"]
    PushBack["push_back(&EntryType)-> bool"]
    PushFront["push_front(&EntryType)-> bool"]
    Remove["remove(&EntryType)-> bool"]
    PopFront["pop_front()-> Option>"]
    InsertAfter["insert_after(&existing, &new)-> bool"]
end
subgraph subGraph0["RawList"]
    RawListStruct["RawList"]
    HeadPtr["head: Option>"]
end

RawListStruct --> Back
RawListStruct --> CursorFront
RawListStruct --> Front
RawListStruct --> HeadPtr
RawListStruct --> InsertAfter
RawListStruct --> IsEmpty
RawListStruct --> Iter
RawListStruct --> PopFront
RawListStruct --> PushBack
RawListStruct --> PushFront
RawListStruct --> Remove
```

Sources: [src/raw_list.rs(L88 - L284)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L88-L284)

## List Operations

### Insertion Operations

The RawList provides several insertion methods that work with raw references and return boolean values indicating success or failure.

|Operation|Method|Safety Requirements|Return Value|
| --- | --- | --- | --- |
|Back Insertion|push_back(&EntryType)|Reference must remain valid while on list|bool- true if inserted, false if already on list|
|Front Insertion|push_front(&EntryType)|Reference must remain valid while on list|bool- true if inserted, false if already on list|
|Positional Insertion|insert_after(&existing, &new)|Both references valid, existing must be on list|bool- true if inserted, false if new already on list|

**Insertion Safety Model**

All insertion operations use the atomic `acquire_for_insertion()` mechanism to prevent double-insertion. The operations are marked `unsafe` because the caller must ensure reference validity for the lifetime of list membership.

Sources: [src/raw_list.rs(L153 - L197)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L153-L197) [src/raw_list.rs(L135 - L151)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L135-L151)

### Removal Operations

Removal operations provide constant-time complexity by updating surrounding pointers directly without traversal.

```mermaid
flowchart TD
subgraph subGraph0["Removal Process"]
    CheckInsertion["Check insertion state(entry.next != None)"]
    UpdatePointers["Update prev.nextand next.prev"]
    UpdateHead["Update head ifremoving head element"]
    ResetLinks["Reset entry.nextand entry.prev to None"]
    ReleaseState["release_after_removal()store(false, Release)"]
end

CheckInsertion --> UpdatePointers
ResetLinks --> ReleaseState
UpdateHead --> ResetLinks
UpdatePointers --> UpdateHead
```

**Constant-Time Removal Implementation**

The `remove_internal()` method achieves O(1) complexity by directly accessing the target element's links rather than traversing the list. The atomic release operation ensures thread-safe state management.

Sources: [src/raw_list.rs(L199 - L245)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L199-L245) [src/raw_list.rs(L247 - L257)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L247-L257)

## Cursor System

The cursor system provides controlled traversal and modification capabilities for the linked list.

### Cursor Types

|Cursor Type|Mutability|Capabilities|
| --- | --- | --- |
|Cursor<'a, G>|Immutable|Read-only traversal, element inspection|
|CursorMut<'a, G>|Mutable|Traversal, element modification, removal|

### Cursor Operations

```mermaid
flowchart TD
subgraph subGraph2["Mutable Cursor"]
    CursorMut["CursorMut<'a, G>"]
    CurrentMut["current() -> Option<&mut EntryType>"]
    RemoveCurrent["remove_current()-> Option>"]
    PeekNext["peek_next() -> Option<&mut EntryType>"]
    PeekPrev["peek_prev() -> Option<&mut EntryType>"]
end
subgraph subGraph1["Read-Only Cursor"]
    Cursor["Cursor<'a, G>"]
    Current["current() -> Option<&EntryType>"]
end
subgraph subGraph0["Cursor Navigation"]
    CommonCursor["CommonCursor"]
    MoveNext["move_next(&RawList)"]
    MovePrev["move_prev(&RawList)"]
    CurrentPos["cur: Option>"]
end

CommonCursor --> CurrentPos
CommonCursor --> MoveNext
CommonCursor --> MovePrev
Cursor --> CommonCursor
Cursor --> Current
CursorMut --> CommonCursor
CursorMut --> CurrentMut
CursorMut --> PeekNext
CursorMut --> PeekPrev
CursorMut --> RemoveCurrent
```

**Cursor Safety Model**

Cursors maintain lifetime relationships with the underlying list to prevent use-after-free scenarios. The mutable cursor provides safe removal operations that automatically advance the cursor position.

Sources: [src/raw_list.rs(L339 - L423)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L339-L423) [src/raw_list.rs(L286 - L329)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L286-L329)

## Iterator Implementation

The iterator system provides both forward and backward traversal capabilities.

### Iterator Structure

```mermaid
flowchart TD
subgraph subGraph2["IntoIterator Support"]
    IntoIter["impl IntoIterator for &RawListinto_iter() -> Iterator"]
end
subgraph subGraph1["Iterator Traits"]
    StdIterator["impl Iteratornext() -> Option<&EntryType>"]
    DoubleEnded["impl DoubleEndedIteratornext_back() -> Option<&EntryType>"]
end
subgraph subGraph0["Iterator Implementation"]
    IteratorStruct["Iterator<'a, G>"]
    CursorFront["cursor_front: Cursor<'a, G>"]
    CursorBack["cursor_back: Cursor<'a, G>"]
end

IntoIter --> IteratorStruct
IteratorStruct --> CursorBack
IteratorStruct --> CursorFront
IteratorStruct --> DoubleEnded
IteratorStruct --> StdIterator
```

**Bidirectional Iteration**

The iterator uses two cursors to enable efficient bidirectional traversal, supporting both `Iterator` and `DoubleEndedIterator` traits for compatibility with standard Rust iteration patterns.

Sources: [src/raw_list.rs(L434 - L464)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L434-L464) [src/raw_list.rs(L425 - L431)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L425-L431)

## Memory Layout and Safety Invariants

### Thread Safety Model

The RawList implements thread safety through atomic operations and careful ordering constraints.

|Component|Thread Safety Mechanism|Ordering|
| --- | --- | --- |
|Insertion Tracking|AtomicBoolwith compare-exchange|Acquireon insert,Relaxedon failure|
|Removal State|AtomicBoolstore operation|Releaseordering|
|Pointer Access|UnsafeCellwith controlled access|Synchronized through insertion state|

**Safety Invariants**

1. **Link Ownership**: Once successfully inserted, the list owns the links until removal
2. **Reference Validity**: Callers must ensure references remain valid while on the list
3. **Single List Membership**: An element can only be on one list at a time (enforced by atomic insertion tracking)
4. **Removal Safety**: Only elements currently on the list may be removed

Sources: [src/raw_list.rs(L40 - L46)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L40-L46) [src/raw_list.rs(L331 - L337)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L331-L337)

### Memory Layout Diagram

```mermaid
flowchart TD
subgraph subGraph3["List Head"]
    RawListHead["RawList.headOption>"]
end
subgraph subGraph2["ListEntry Pointers"]
    NextPointer["next: Option>(8 bytes on 64-bit)"]
    PrevPointer["prev: Option>(8 bytes on 64-bit)"]
end
subgraph subGraph1["Links Internal Layout"]
    AtomicBool["inserted: AtomicBool(1 byte + padding)"]
    UnsafeCell["entry: UnsafeCell(contains raw pointers)"]
end
subgraph subGraph0["Node Memory Structure"]
    UserData["User Data Fields(application-specific)"]
    LinksField["links: Links"]
end

LinksField --> AtomicBool
LinksField --> UnsafeCell
NextPointer --> UserData
PrevPointer --> UserData
RawListHead --> UserData
UnsafeCell --> NextPointer
UnsafeCell --> PrevPointer
UserData --> LinksField
```

Sources: [src/raw_list.rs(L35 - L38)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L35-L38) [src/raw_list.rs(L74 - L77)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L74-L77) [src/raw_list.rs(L93 - L95)&emsp;](https://github.com/arceos-org/linked_list_r4l/blob/353828c1/src/raw_list.rs#L93-L95)