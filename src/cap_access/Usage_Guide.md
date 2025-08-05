# Usage Guide

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/cap_access/blob/ad71552e/README.md)
> * [src/lib.rs](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs)

This guide provides practical examples and patterns for using the `cap_access` library in real applications. It covers the essential usage patterns, access methods, and best practices for implementing capability-based access control in your code.

For architectural details about the underlying capability system, see [Core Architecture](/arceos-org/cap_access/2-core-architecture). For information about integrating with ArceOS, see [ArceOS Integration](/arceos-org/cap_access/4-arceos-integration).

## Basic Usage Patterns

The fundamental workflow with `cap_access` involves creating protected objects with `WithCap::new()` and accessing them through capability-checked methods.

### Creating Protected Objects

```mermaid
flowchart TD
subgraph subGraph0["Cap Constants"]
    READ["Cap::READ"]
    WRITE["Cap::WRITE"]
    EXECUTE["Cap::EXECUTE"]
end
Object["Raw Object(T)"]
WithCap_new["WithCap::new()"]
Capability["Cap Permission(READ | WRITE | EXECUTE)"]
Protected["WithCap<T>Protected Object"]

Capability --> WithCap_new
EXECUTE --> Capability
Object --> WithCap_new
READ --> Capability
WRITE --> Capability
WithCap_new --> Protected
```

The most common pattern is to wrap objects at creation time with appropriate capabilities:

|Object Type|Typical Capabilities|Use Case|
| --- | --- | --- |
|Configuration Data|Cap::READ|Read-only system settings|
|User Files|Cap::READ \| Cap::WRITE|Editable user content|
|Executable Code|Cap::READ \| Cap::EXECUTE|Program binaries|
|System Resources|Cap::READ \| Cap::WRITE \| Cap::EXECUTE|Full access resources|

Sources: [src/lib.rs(L4 - L15)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L4-L15) [src/lib.rs(L23 - L27)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L23-L27)

### Capability Composition

Capabilities can be combined using bitwise operations to create complex permission sets:

```mermaid
flowchart TD
subgraph subGraph1["Combined Capabilities"]
    RW["Cap::READ | Cap::WRITERead-Write Access"]
    RX["Cap::READ | Cap::EXECUTERead-Execute Access"]
    RWX["Cap::READ | Cap::WRITE | Cap::EXECUTEFull Access"]
end
subgraph subGraph0["Individual Capabilities"]
    READ_CAP["Cap::READ(1 << 0)"]
    WRITE_CAP["Cap::WRITE(1 << 1)"]
    EXEC_CAP["Cap::EXECUTE(1 << 2)"]
end

EXEC_CAP --> RWX
EXEC_CAP --> RX
READ_CAP --> RW
READ_CAP --> RWX
READ_CAP --> RX
WRITE_CAP --> RW
WRITE_CAP --> RWX
```

Sources: [src/lib.rs(L4 - L15)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L4-L15) [README.md(L16 - L29)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/README.md#L16-L29)

## Access Method Comparison

The `WithCap<T>` struct provides three distinct access patterns, each with different safety and error handling characteristics:

```mermaid
flowchart TD
subgraph subGraph1["Return Types"]
    Bool["Boolean CheckValidation only"]
    Option["Option PatternNone on failure"]
    Result["Result PatternCustom error on failure"]
    Direct["Direct ReferenceNo safety checks"]
end
subgraph subGraph0["Access Methods"]
    can_access["can_access(cap)→ bool"]
    access["access(cap)→ Option<&T>"]
    access_or_err["access_or_err(cap, err)→ Result<&T, E>"]
    access_unchecked["access_unchecked()→ &T (UNSAFE)"]
end
WithCap["WithCap<T>Protected Object"]

WithCap --> access
WithCap --> access_or_err
WithCap --> access_unchecked
WithCap --> can_access
access --> Option
access_or_err --> Result
access_unchecked --> Direct
can_access --> Bool
```

### Method Selection Guidelines

|Method|When to Use|Safety Level|Performance|
| --- | --- | --- | --- |
|can_access()|Pre-flight validation checks|Safe|Fastest|
|access()|Optional access patterns|Safe|Fast|
|access_or_err()|Error handling with context|Safe|Fast|
|access_unchecked()|Performance-critical paths|Unsafe|Fastest|

Sources: [src/lib.rs(L34 - L48)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L34-L48) [src/lib.rs(L59 - L78)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L59-L78) [src/lib.rs(L80 - L99)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L80-L99) [src/lib.rs(L50 - L57)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L50-L57)

## Practical Access Patterns

### Safe Access with Option Handling

The `access()` method returns `Option<&T>` and is ideal for scenarios where access failure is expected and should be handled gracefully:

```mermaid
sequenceDiagram
    participant Client as Client
    participant WithCapaccess as "WithCap::access()"
    participant WithCapcan_access as "WithCap::can_access()"
    participant innerT as "inner: T"

    Client ->> WithCapaccess: access(requested_cap)
    WithCapaccess ->> WithCapcan_access: self.cap.contains(requested_cap)
    alt Capability Match
        WithCapcan_access -->> WithCapaccess: true
        WithCapaccess ->> innerT: &self.inner
        innerT -->> WithCapaccess: &T
        WithCapaccess -->> Client: Some(&T)
    else Capability Mismatch
        WithCapcan_access -->> WithCapaccess: false
        WithCapaccess -->> Client: None
    end
```

This pattern is demonstrated in the README examples where checking for `Cap::EXECUTE` on a read-write object returns `None`.

Sources: [src/lib.rs(L72 - L78)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L72-L78) [README.md(L22 - L28)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/README.md#L22-L28)

### Error-Based Access with Context

The `access_or_err()` method provides custom error messages for better debugging and user feedback:

```mermaid
flowchart TD
subgraph Output["Output"]
    Result_Ok["Result::Ok(&T)"]
    Result_Err["Result::Err(E)"]
end
subgraph subGraph1["access_or_err Process"]
    Check["can_access(cap)"]
    Success["Ok(&inner)"]
    Failure["Err(custom_error)"]
end
subgraph Input["Input"]
    Cap_Request["Requested Capability"]
    Error_Value["Custom Error Value"]
end

Cap_Request --> Check
Check --> Failure
Check --> Success
Error_Value --> Check
Failure --> Result_Err
Success --> Result_Ok
```

This method is particularly useful in system programming contexts where specific error codes or messages are required for debugging.

Sources: [src/lib.rs(L93 - L99)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L93-L99)

### Unsafe High-Performance Access

The `access_unchecked()` method bypasses capability validation for performance-critical code paths:

**Safety Requirements:**

* Caller must manually verify capability compliance
* Should only be used in trusted, performance-critical contexts
* Requires `unsafe` block, making the safety contract explicit

Sources: [src/lib.rs(L55 - L57)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L55-L57)

## Best Practices

### Capability Design Patterns

1. **Principle of Least Privilege**: Grant minimal necessary capabilities
2. **Capability Composition**: Use bitwise OR to combine permissions
3. **Validation at Boundaries**: Check capabilities at system boundaries
4. **Consistent Error Handling**: Choose one access method per subsystem

### Integration Patterns

```mermaid
flowchart TD
subgraph subGraph2["Protected Resources"]
    FileSystem["File System Objects"]
    Memory["Memory Regions"]
    DeviceDrivers["Device Drivers"]
end
subgraph subGraph1["cap_access Integration"]
    WithCap_Factory["WithCap CreationWithCap::new(obj, cap)"]
    Access_Layer["Access Controlaccess() / access_or_err()"]
    Validation["Capability Validationcan_access()"]
end
subgraph subGraph0["Application Layer"]
    UserCode["User Application Code"]
    ServiceLayer["Service Layer"]
end

Access_Layer --> Validation
ServiceLayer --> Access_Layer
ServiceLayer --> WithCap_Factory
UserCode --> Access_Layer
UserCode --> WithCap_Factory
WithCap_Factory --> DeviceDrivers
WithCap_Factory --> FileSystem
WithCap_Factory --> Memory
```

### Common Usage Scenarios

|Scenario|Recommended Pattern|Example|
| --- | --- | --- |
|File Access Control|WithCap::new(file, Cap::READ \| Cap::WRITE)|User file permissions|
|Memory Protection|WithCap::new(region, Cap::READ \| Cap::EXECUTE)|Code segment protection|
|Device Driver Access|WithCap::new(device, Cap::WRITE)|Hardware write-only access|
|Configuration Data|WithCap::new(config, Cap::READ)|Read-only system settings|

Sources: [src/lib.rs(L23 - L27)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L23-L27) [README.md(L19 - L24)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/README.md#L19-L24)

## Advanced Usage Patterns

### Dynamic Capability Checking

The `can_access()` method enables sophisticated access control logic:

```

```

This pattern is essential for security-sensitive applications that need comprehensive access logging and audit trails.

Sources: [src/lib.rs(L46 - L48)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L46-L48)

### Multi-Level Access Control

Complex systems often require nested capability checks across multiple abstraction layers:

```mermaid
flowchart TD
subgraph subGraph1["WithCap Instances"]
    AppData["WithCap<UserData>"]
    KernelData["WithCap<SystemCall>"]
    HWData["WithCap<Device>"]
end
subgraph subGraph0["System Architecture"]
    App["Application LayerCap::READ | Cap::WRITE"]
    Kernel["Kernel LayerCap::EXECUTE"]
    Hardware["Hardware LayerCap::READ | Cap::WRITE | Cap::EXECUTE"]
end
UserAccess["User Read Access"]
SyscallAccess["System Call Execution"]
DirectHW["Direct Hardware Access"]

App --> AppData
AppData --> UserAccess
HWData --> DirectHW
Hardware --> HWData
Kernel --> KernelData
KernelData --> SyscallAccess
```

This layered approach ensures that capability validation occurs at appropriate system boundaries while maintaining performance where needed.

Sources: [src/lib.rs(L1 - L101)&emsp;](https://github.com/arceos-org/cap_access/blob/ad71552e/src/lib.rs#L1-L101)