# Usage Guide and Examples

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/cpumask/blob/a7cfa639/README.md)
> * [src/lib.rs](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs)

This page provides practical examples and common usage patterns for the `cpumask` library in operating system contexts. It demonstrates how to effectively use the `CpuMask<SIZE>` struct for CPU affinity management, process scheduling, and system resource allocation.

For detailed API documentation, see [API Reference](/arceos-org/cpumask/2-api-reference). For implementation details and performance characteristics, see [Architecture and Design](/arceos-org/cpumask/3-architecture-and-design).

## Basic Usage Patterns

### Creating CPU Masks

The most common way to start working with CPU masks is through the various construction methods provided by `CpuMask<SIZE>`:

```mermaid
flowchart TD
A["CpuMask::new()"]
B["Empty mask - all false"]
C["CpuMask::full()"]
D["Full mask - all true"]
E["CpuMask::mask(bits)"]
F["Range mask - first bits true"]
G["CpuMask::one_shot(index)"]
H["Single CPU mask"]
I["CpuMask::from_raw_bits(value)"]
J["From usize value"]
K["CpuMask::from_value(data)"]
L["From backing store type"]
M["Basic CPU set operations"]

A --> B
B --> M
C --> D
D --> M
E --> F
F --> M
G --> H
H --> M
I --> J
J --> M
K --> L
L --> M
```

Sources: [src/lib.rs(L72 - L128)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L72-L128)

### Basic Manipulation Operations

The fundamental operations for working with individual CPU bits follow a consistent pattern:

|Operation|Method|Description|Return Value|
| --- | --- | --- | --- |
|Test bit|get(index)|Check if CPU is in mask|bool|
|Set bit|set(index, value)|Add/remove CPU from mask|Previousboolvalue|
|Count bits|len()|Number of CPUs in mask|usize|
|Test empty|is_empty()|Check if no CPUs set|bool|
|Test full|is_full()|Check if all CPUs set|bool|

Sources: [src/lib.rs(L148 - L179)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L148-L179)

### Finding CPUs in Masks

The library provides efficient methods for locating specific CPUs within masks:

```mermaid
flowchart TD
A["CpuMask"]
B["first_index()"]
C["last_index()"]
D["next_index(current)"]
E["prev_index(current)"]
F["Inverse operations"]
G["first_false_index()"]
H["last_false_index()"]
I["next_false_index(current)"]
J["prev_false_index(current)"]
K["Option<usize>"]

A --> B
A --> C
A --> D
A --> E
B --> K
C --> K
D --> K
E --> K
F --> G
F --> H
F --> I
F --> J
G --> K
H --> K
I --> K
J --> K
```

Sources: [src/lib.rs(L182 - L228)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L182-L228)

## Set Operations and Logic

### Bitwise Operations

CPU masks support standard bitwise operations for combining and manipulating sets of CPUs:

```mermaid
flowchart TD
A["mask1: CpuMask<N>"]
E["mask1 & mask2"]
B["mask2: CpuMask<N>"]
F["Intersection - CPUs in both"]
G["mask1 | mask2"]
H["Union - CPUs in either"]
I["mask1 ^ mask2"]
J["Symmetric difference - CPUs in one but not both"]
K["!mask1"]
L["Complement - inverse of mask"]
M["Assignment variants"]
N["&=, |=, ^="]
O["In-place operations"]

A --> E
A --> G
A --> I
A --> K
B --> E
B --> G
B --> I
E --> F
G --> H
I --> J
K --> L
M --> N
N --> O
```

Sources: [src/lib.rs(L253 - L326)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L253-L326)

### Practical Set Operations Example

Consider a scenario where you need to manage CPU allocation for different process types:

1. **System processes** - CPUs 0-3 (reserved for kernel)
2. **User processes** - CPUs 4-15 (general workload)
3. **Real-time processes** - CPUs 12-15 (high-performance subset)

This demonstrates how set operations enable sophisticated CPU management policies.

Sources: [src/lib.rs(L253 - L287)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L253-L287)

## Iteration and Traversal

### Forward and Backward Iteration

The `CpuMask` implements both `Iterator` and `DoubleEndedIterator`, enabling flexible traversal patterns:

```

```

Sources: [src/lib.rs(L237 - L518)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L237-L518)

### Iteration Use Cases

The iterator yields CPU indices where the bit is set to `true`, making it ideal for:

* **Process scheduling**: Iterate through available CPUs for task assignment
* **Load balancing**: Traverse CPU sets to find optimal placement
* **Resource monitoring**: Check specific CPU states in sequence
* **Affinity enforcement**: Validate process can run on assigned CPUs

Sources: [src/lib.rs(L412 - L427)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L412-L427)

## Operating System Scenarios

### Process Scheduling Workflow

This diagram shows how `CpuMask` integrates into a typical process scheduler:

```mermaid
flowchart TD
A["Process creation"]
B["Determine CPU affinity policy"]
C["CpuMask::new() or policy-specific constructor"]
D["Store affinity in process control block"]
E["Scheduler activation"]
F["Load process affinity mask"]
G["available_cpus & process_affinity"]
H["Any CPUs available?"]
I["Select CPU from mask"]
J["Wait or migrate"]
K["Schedule process on selected CPU"]
L["Update available_cpus"]
M["CPU hotplug event"]
N["Update system CPU mask"]
O["Revalidate all process affinities"]
P["Migrate processes if needed"]

A --> B
B --> C
C --> D
E --> F
F --> G
G --> H
H --> I
H --> J
I --> K
J --> L
L --> G
M --> N
N --> O
O --> P
```

Sources: [src/lib.rs(L9 - L23)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L9-L23)

### NUMA-Aware CPU Management

In NUMA systems, CPU masks help maintain memory locality:

```mermaid
flowchart TD
subgraph subGraph1["NUMA Node 1"]
    C["CPUs 8-15"]
    D["Local Memory"]
end
subgraph subGraph0["NUMA Node 0"]
    A["CPUs 0-7"]
    B["Local Memory"]
end
E["Process A"]
F["CpuMask::mask(8) - Node 0 only"]
G["Process B"]
H["CpuMask::from_raw_bits(0xFF00) - Node 1 only"]
I["System Process"]
J["CpuMask::full() - Any CPU"]
K["Load balancer decides"]

E --> F
F --> A
G --> H
H --> C
I --> J
J --> K
K --> A
K --> C
```

Sources: [src/lib.rs(L86 - L94)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L86-L94) [src/lib.rs(L106 - L119)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L106-L119)

### Thread Affinity Management

For multi-threaded applications, CPU masks enable fine-grained control:

|Thread Type|Affinity Strategy|Implementation|
| --- | --- | --- |
|Main thread|Single CPU|CpuMask::one_shot(0)|
|Worker threads|Subset of CPUs|CpuMask::mask(worker_count)|
|I/O threads|Non-main CPUs|!CpuMask::one_shot(0)|
|RT threads|Dedicated CPUs|CpuMask::from_raw_bits(rt_mask)|

Sources: [src/lib.rs(L121 - L128)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L121-L128) [src/lib.rs(L289 - L299)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L289-L299)

## Advanced Usage Patterns

### Dynamic CPU Set Management

Real systems require dynamic adjustment of CPU availability:

```

```

Sources: [src/lib.rs(L177 - L180)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L177-L180) [src/lib.rs(L301 - L326)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L301-L326)

### Performance Optimization Strategies

The choice of `SIZE` parameter significantly impacts performance:

|System Size|Recommended SIZE|Storage Type|Use Case|
| --- | --- | --- | --- |
|Single core|SIZE = 1|bool|Embedded systems|
|Small SMP (≤8)|SIZE = 8|u8|IoT devices|
|Desktop/Server (≤32)|SIZE = 32|u32|General purpose|
|Large server (≤128)|SIZE = 128|u128|HPC systems|
|Massive systems|SIZE = 1024|[u128; 8]|Cloud infrastructure|

The automatic storage selection in [src/lib.rs(L12 - L16)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L12-L16) ensures optimal memory usage and performance characteristics for each size category.

### Error Handling Patterns

While `CpuMask` operations are generally infallible, certain usage patterns require validation:

* **Bounds checking**: Methods use `debug_assert!` for index validation
* **Size validation**: `from_raw_bits()` asserts value fits in SIZE bits
* **Range validation**: `one_shot()` panics if index exceeds SIZE

In production systems, wrap these operations with appropriate error handling based on your safety requirements.

Sources: [src/lib.rs(L106 - L108)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L106-L108) [src/lib.rs(L123 - L124)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L123-L124) [src/lib.rs(L168 - L170)&emsp;](https://github.com/arceos-org/cpumask/blob/a7cfa639/src/lib.rs#L168-L170)