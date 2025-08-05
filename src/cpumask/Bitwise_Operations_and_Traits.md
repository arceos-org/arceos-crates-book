# Bitwise Operations and Traits

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs)

This page documents the operator overloads, trait implementations, and set operations available on `CpuMask` instances. These operations enable intuitive mathematical set operations, comparisons, and conversions between CPU mask representations.

For information about basic construction and conversion methods, see [Construction and Conversion Methods](/arceos-org/cpumask/2.1-construction-and-conversion-methods). For details about query operations and bit manipulation, see [Query and Inspection Operations](/arceos-org/cpumask/2.2-query-and-inspection-operations).

## Bitwise Set Operations

The `CpuMask` type implements standard bitwise operators that correspond to mathematical set operations on CPU collections.

### Binary Operators

The `CpuMask` struct implements four core binary bitwise operations:

|Operator|Trait|Set Operation|Description|
| --- | --- | --- | --- |
|&|BitAnd|Intersection|CPUs present in both masks|
|\||BitOr|Union|CPUs present in either mask|
|^|BitXor|Symmetric Difference|CPUs present in exactly one mask|
|!|Not|Complement|Invert all CPU bits|

Each operator returns a new `CpuMask` instance without modifying the operands.

```mermaid
flowchart TD
subgraph subGraph1["Implementation Details"]
    J["self.value.bitand(rhs.value)"]
    K["self.value.bitor(rhs.value)"]
    L["self.value.bitxor(rhs.value)"]
    M["self.value.not()"]
end
subgraph subGraph0["Bitwise Operations"]
    A["CpuMask"]
    B["& (BitAnd)"]
    C["| (BitOr)"]
    D["^ (BitXor)"]
    E["! (Not)"]
    F["Intersection"]
    G["Union"]
    H["Symmetric Difference"]
    I["Complement"]
end

A --> B
A --> C
A --> D
A --> E
B --> F
C --> G
D --> H
E --> I
F --> J
G --> K
H --> L
I --> M
```

**Sources:** [src/lib.rs(L253 - L299)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L253-L299)

### Assignment Operators

The `CpuMask` type also implements in-place assignment variants for efficiency:

|Operator|Trait|Description|
| --- | --- | --- |
|&=|BitAndAssign|In-place intersection|
|\|=|BitOrAssign|In-place union|
|^=|BitXorAssign|In-place symmetric difference|

These operators modify the left operand directly, avoiding temporary allocations.

```mermaid
flowchart TD
subgraph subGraph0["Assignment Operations"]
    A["cpumask1 &= cpumask2"]
    B["BitAndAssign::bitand_assign"]
    C["cpumask1 |= cpumask2"]
    D["BitOrAssign::bitor_assign"]
    E["cpumask1 ^= cpumask2"]
    F["BitXorAssign::bitxor_assign"]
    G["self.value.bitand_assign(rhs.value)"]
    H["self.value.bitor_assign(rhs.value)"]
    I["self.value.bitxor_assign(rhs.value)"]
end

A --> B
B --> G
C --> D
D --> H
E --> F
F --> I
```

**Sources:** [src/lib.rs(L301 - L326)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L301-L326)

## Comparison and Ordering Traits

The `CpuMask` type implements a complete hierarchy of comparison traits, enabling sorting and ordering operations on CPU mask collections.

### Trait Implementation Hierarchy

```mermaid
flowchart TD
subgraph subGraph1["Implementation Details"]
    F["self.value.as_value().partial_cmp(other.value.as_value())"]
    G["self.value.as_value().cmp(other.value.as_value())"]
end
subgraph subGraph0["Comparison Traits"]
    A["PartialEq"]
    B["Eq"]
    C["PartialOrd"]
    D["Ord"]
    E["CpuMask implements"]
end

A --> B
B --> C
C --> D
C --> F
D --> G
E --> A
E --> B
E --> C
E --> D
```

The comparison operations delegate to the underlying storage type's comparison methods, ensuring consistent ordering across different storage implementations.

**Sources:** [src/lib.rs(L17)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L17-L17) [src/lib.rs(L48 - L66)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L48-L66)

## Utility Traits

### Debug Formatting

The `Debug` trait provides a human-readable representation showing the indices of set CPU bits:

```

```

**Sources:** [src/lib.rs(L25 - L36)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L25-L36)

### Hash Implementation

The `Hash` trait enables use of `CpuMask` instances as keys in hash-based collections:

```mermaid
flowchart TD
A["Hash::hash"]
B["self.value.as_value().hash(state)"]
C["Delegates to underlying storage hash"]

A --> B
B --> C
```

The hash implementation requires that the underlying storage type implement `Hash`, which is satisfied by all primitive integer types used for storage.

**Sources:** [src/lib.rs(L38 - L46)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L38-L46)

### Derived Traits

The `CpuMask` struct automatically derives several fundamental traits:

|Trait|Purpose|
| --- | --- |
|Clone|Creates deep copies of CPU masks|
|Copy|Enables pass-by-value for smaller masks|
|Default|Creates empty CPU masks|
|Eq|Enables exact equality comparisons|
|PartialEq|Enables equality comparisons|

**Sources:** [src/lib.rs(L17)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L17-L17)

## Conversion Traits

The library provides specialized `From` trait implementations for converting between `CpuMask` instances and `u128` arrays for larger mask sizes.

### Array Conversion Support

```mermaid
flowchart TD
subgraph subGraph1["From CpuMask to Array"]
    I["[u128; 2]"]
    J["[u128; 3]"]
    K["[u128; 4]"]
    L["[u128; 8]"]
end
subgraph subGraph0["From Array to CpuMask"]
    A["[u128; 2]"]
    B["CpuMask<256>"]
    C["[u128; 3]"]
    D["CpuMask<384>"]
    E["[u128; 4]"]
    F["CpuMask<512>"]
    G["[u128; 8]"]
    H["CpuMask<1024>"]
end

A --> B
B --> I
C --> D
D --> J
E --> F
F --> K
G --> H
H --> L
```

These implementations enable seamless conversion between raw array representations and type-safe CPU mask instances for masks requiring more than 128 bits of storage.

**Sources:** [src/lib.rs(L328 - L410)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L328-L410)

## Iterator Trait Implementation

The `IntoIterator` trait enables `CpuMask` instances to be used directly in `for` loops and with iterator methods.

### Iterator Implementation Structure

```mermaid
flowchart TD
subgraph subGraph1["Internal State"]
    H["head: Option"]
    I["tail: Option"]
    J["data: &'a CpuMask"]
end
subgraph subGraph0["Iterator Pattern"]
    A["&CpuMask"]
    B["IntoIterator::into_iter"]
    C["Iter<'a, SIZE>"]
    D["Iterator::next"]
    E["DoubleEndedIterator::next_back"]
    F["Returns Option"]
    G["Returns Option"]
end

A --> B
B --> C
C --> D
C --> E
C --> H
C --> I
C --> J
D --> F
E --> G
```

The iterator yields the indices of set bits as `usize` values, enabling direct access to CPU indices without needing to manually scan the mask.

**Sources:** [src/lib.rs(L237 - L251)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L237-L251) [src/lib.rs(L412 - L518)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L412-L518)