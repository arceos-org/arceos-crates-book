# Capability System

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs)

## Purpose and Scope

The Capability System forms the foundation of access control in the cap_access library. This document covers the `Cap` bitflags structure, the three fundamental capability types (READ, WRITE, EXECUTE), and the mechanisms for combining and checking capabilities. For information about how capabilities are used with protected objects, see [Object Protection with WithCap](/arceos-org/cap_access/2.2-object-protection-with-withcap). For details on access control methods that utilize these capabilities, see [Access Control Methods](/arceos-org/cap_access/2.3-access-control-methods).

## Cap Bitflags Structure

The capability system is implemented using the `bitflags` crate to provide efficient bit-level operations for permission management. The `Cap` structure represents unforgeable access tokens that can be combined using bitwise operations.

```mermaid
flowchart TD
subgraph subGraph2["Bitwise Operations"]
    UNION["Union (|)"]
    INTERSECTION["Intersection (&)"]
    CONTAINS["contains()"]
end
subgraph subGraph1["Bitflags Traits"]
    DEFAULT["Default"]
    DEBUG["Debug"]
    CLONE["Clone"]
    COPY["Copy"]
end
subgraph subGraph0["Cap Structure [lines 4-15]"]
    CAP["Cap: u32"]
    READ["READ = 1 << 0"]
    WRITE["WRITE = 1 << 1"]
    EXECUTE["EXECUTE = 1 << 2"]
end

CAP --> CLONE
CAP --> COPY
CAP --> DEBUG
CAP --> DEFAULT
CAP --> EXECUTE
CAP --> READ
CAP --> WRITE
EXECUTE --> UNION
INTERSECTION --> CONTAINS
READ --> UNION
UNION --> CONTAINS
WRITE --> UNION
```

**Cap Bitflags Architecture**

The `Cap` struct is defined as a bitflags structure that wraps a `u32` value, providing type-safe capability manipulation with automatic trait derivations for common operations.

Sources: [src/lib.rs(L4 - L15)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L4-L15)

## Basic Capability Types

The capability system defines three fundamental access rights that correspond to common operating system permissions:

|Capability|Bit Position|Value|Purpose|
| --- | --- | --- | --- |
|READ|0|1 << 0(1)|Grants readable access to protected data|
|WRITE|1|1 << 1(2)|Grants writable access to protected data|
|EXECUTE|2|1 << 2(4)|Grants executable access to protected data|

```mermaid
flowchart TD
subgraph subGraph1["Capability Constants"]
    READ_CONST["Cap::READ = 0x01"]
    WRITE_CONST["Cap::WRITE = 0x02"]
    EXECUTE_CONST["Cap::EXECUTE = 0x04"]
end
subgraph subGraph0["Bit Layout [u32]"]
    BIT2["Bit 2: EXECUTE"]
    BIT1["Bit 1: WRITE"]
    BIT0["Bit 0: READ"]
    UNUSED["Bits 3-31: Reserved"]
end

BIT0 --> READ_CONST
BIT1 --> WRITE_CONST
BIT2 --> EXECUTE_CONST
```

**Capability Bit Layout**

Each capability type occupies a specific bit position, allowing for efficient combination and checking operations using bitwise arithmetic.

Sources: [src/lib.rs(L8 - L14)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L8-L14)

## Capability Combinations

Capabilities can be combined using bitwise OR operations to create composite permissions. The bitflags implementation provides natural syntax for capability composition:

```mermaid
flowchart TD
subgraph subGraph2["Use Cases"]
    READONLY["Read-only files"]
    READWRITE["Mutable data"]
    EXECUTABLE["Program files"]
    FULLACCESS["Administrative access"]
end
subgraph subGraph1["Common Combinations"]
    RW["Cap::READ | Cap::WRITE"]
    RX["Cap::READ | Cap::EXECUTE"]
    WX["Cap::WRITE | Cap::EXECUTE"]
    RWX["Cap::READ | Cap::WRITE | Cap::EXECUTE"]
end
subgraph subGraph0["Single Capabilities"]
    R["Cap::READ"]
    W["Cap::WRITE"]
    X["Cap::EXECUTE"]
end

R --> READONLY
R --> RW
R --> RWX
R --> RX
RW --> READWRITE
RWX --> FULLACCESS
RX --> EXECUTABLE
W --> RW
W --> RWX
W --> WX
X --> RWX
X --> RX
X --> WX
```

**Capability Combination Patterns**

The bitflags design enables natural combination of capabilities to express complex permission requirements.

Sources: [src/lib.rs(L4 - L15)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L4-L15)

## Capability Checking Logic

The capability system provides the `contains` method for checking whether a set of capabilities includes required permissions. This forms the foundation for all access control decisions in the system.

```mermaid
flowchart TD
subgraph subGraph0["contains() Logic"]
    BITWISE["(self.bits & other.bits) == other.bits"]
    SUBSET["Check if 'other' is subset of 'self'"]
end
subgraph subGraph1["Example Scenarios"]
    SCENARIO1["Object has READ|WRITERequest: READ → true"]
    SCENARIO2["Object has READRequest: WRITE → false"]
    SCENARIO3["Object has READ|WRITERequest: READ|WRITE → true"]
end
START["can_access(cap: Cap)"]
CHECK["self.cap.contains(cap)"]
TRUE["Return true"]
FALSE["Return false"]

BITWISE --> SUBSET
CHECK --> BITWISE
START --> CHECK
SUBSET --> FALSE
SUBSET --> TRUE
```

**Capability Checking Flow**

The `can_access` method uses bitwise operations to determine if the requested capabilities are a subset of the available capabilities.

Sources: [src/lib.rs(L46 - L48)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L46-L48)

## Implementation Details

The `Cap` structure leverages several key design patterns for efficient and safe capability management:

### Bitflags Integration

The implementation uses the `bitflags!` macro to generate a complete capability management API, including:

* Automatic implementation of bitwise operations (`|`, `&`, `^`, `!`)
* Type-safe flag manipulation methods
* Built-in `contains()` method for subset checking
* Standard trait implementations (`Debug`, `Clone`, `Copy`, `Default`)

### Memory Efficiency

The `Cap` structure occupies only 4 bytes (`u32`) regardless of capability combinations, making it suitable for embedded environments where memory usage is critical.

### Constant Evaluation

The `can_access` method is marked as `const fn`, enabling compile-time capability checking when capability values are known at compile time.

Sources: [src/lib.rs(L4 - L15)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L4-L15) [src/lib.rs(L46 - L48)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L46-L48)