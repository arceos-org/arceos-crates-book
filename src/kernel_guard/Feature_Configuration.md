# Feature Configuration

> **Relevant source files**
> * [Cargo.toml](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/Cargo.toml)
> * [src/lib.rs](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/lib.rs)

This document explains the feature-based configuration system in the `kernel_guard` crate, specifically focusing on the `preempt` feature and its impact on guard availability and functionality. For information about implementing the required traits when using features, see [Implementing KernelGuardIf](/arceos-org/kernel_guard/4.2-implementing-kernelguardif).

## Overview

The `kernel_guard` crate uses Cargo features to provide conditional functionality that adapts to different system requirements. The primary feature is `preempt`, which enables kernel preemption control capabilities in guard implementations. This feature system allows the crate to be used in both simple interrupt-only scenarios and more complex preemptive kernel environments.

## Feature Definitions

The crate defines its features in the Cargo manifest with a minimal configuration approach:

|Feature|Default|Purpose|
| --- | --- | --- |
|preempt|Disabled|Enables kernel preemption control in applicable guards|
|default|Empty|No features enabled by default|

**Sources**: [Cargo.toml(L14 - L16)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/Cargo.toml#L14-L16)

## Preempt Feature Behavior

### Feature Impact on Guard Implementations

The `preempt` feature primarily affects two guard types: `NoPreempt` and `NoPreemptIrqSave`. When this feature is enabled, these guards will attempt to disable and re-enable kernel preemption around critical sections.

#### Compilation-Time Feature Resolution

```mermaid
flowchart TD
COMPILE["Compilation Start"]
FEATURE_CHECK["preempt featureenabled?"]
PREEMPT_ENABLED["Preemption ControlActive"]
PREEMPT_DISABLED["Preemption ControlNo-Op"]
NOPRE_IMPL["NoPreempt::acquire()calls KernelGuardIf::disable_preempt"]
NOPRE_REL["NoPreempt::release()calls KernelGuardIf::enable_preempt"]
NOPRE_NOOP["NoPreempt::acquire()no operation"]
NOPRE_NOOP_REL["NoPreempt::release()no operation"]
COMBO_IMPL["NoPreemptIrqSave::acquire()calls disable_preempt + IRQ disable"]
COMBO_REL["NoPreemptIrqSave::release()IRQ restore + enable_preempt"]
COMBO_IRQ["NoPreemptIrqSave::acquire()IRQ disable only"]
COMBO_IRQ_REL["NoPreemptIrqSave::release()IRQ restore only"]

COMPILE --> FEATURE_CHECK
FEATURE_CHECK --> PREEMPT_DISABLED
FEATURE_CHECK --> PREEMPT_ENABLED
PREEMPT_DISABLED --> COMBO_IRQ
PREEMPT_DISABLED --> COMBO_IRQ_REL
PREEMPT_DISABLED --> NOPRE_NOOP
PREEMPT_DISABLED --> NOPRE_NOOP_REL
PREEMPT_ENABLED --> COMBO_IMPL
PREEMPT_ENABLED --> COMBO_REL
PREEMPT_ENABLED --> NOPRE_IMPL
PREEMPT_ENABLED --> NOPRE_REL
```

**Sources**: [src/lib.rs(L149 - L161)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/lib.rs#L149-L161) [src/lib.rs(L163 - L179)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/lib.rs#L163-L179)

### Guard Behavior Matrix

The following table shows how each guard type behaves based on feature configuration:

|Guard Type|preempt Feature Disabled|preempt Feature Enabled|
| --- | --- | --- |
|NoOp|No operation|No operation|
|IrqSave|IRQ disable/restore only|IRQ disable/restore only|
|NoPreempt|No operation|Preemption disable/enable viaKernelGuardIf|
|NoPreemptIrqSave|IRQ disable/restore only|Preemption disable + IRQ disable/restore|

### Implementation Details

#### NoPreempt Guard Feature Integration

```

```

**Sources**: [src/lib.rs(L149 - L161)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/lib.rs#L149-L161) [src/lib.rs(L200 - L217)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/lib.rs#L200-L217)

#### NoPreemptIrqSave Combined Behavior

The `NoPreemptIrqSave` guard demonstrates the most complex feature-dependent behavior, combining both IRQ control and optional preemption control:

```mermaid
sequenceDiagram
    participant UserCode as "User Code"
    participant NoPreemptIrqSave as "NoPreemptIrqSave"
    participant KernelGuardIf as "KernelGuardIf"
    participant archlocal_irq_ as "arch::local_irq_*"

    UserCode ->> NoPreemptIrqSave: new()
    NoPreemptIrqSave ->> NoPreemptIrqSave: acquire()
    alt preempt feature enabled
        NoPreemptIrqSave ->> KernelGuardIf: disable_preempt()
        Note over KernelGuardIf: Kernel preemption disabled
    else preempt feature disabled
        Note over NoPreemptIrqSave: Skip preemption control
    end
    NoPreemptIrqSave ->> archlocal_irq_: local_irq_save_and_disable()
    archlocal_irq_ -->> NoPreemptIrqSave: irq_state
    Note over NoPreemptIrqSave,archlocal_irq_: Critical section active
    UserCode ->> NoPreemptIrqSave: drop()
    NoPreemptIrqSave ->> NoPreemptIrqSave: release(irq_state)
    NoPreemptIrqSave ->> archlocal_irq_: local_irq_restore(irq_state)
    alt preempt feature enabled
        NoPreemptIrqSave ->> KernelGuardIf: enable_preempt()
        Note over KernelGuardIf: Kernel preemption restored
    else preempt feature disabled
        Note over NoPreemptIrqSave: Skip preemption control
    end
```

**Sources**: [src/lib.rs(L163 - L179)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/lib.rs#L163-L179) [src/lib.rs(L220 - L237)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/lib.rs#L220-L237)

## Integration Requirements

### Feature Dependencies

When the `preempt` feature is enabled, user code must provide an implementation of the `KernelGuardIf` trait using the `crate_interface` mechanism. The relationship between feature activation and implementation requirements is:

|Configuration|User Implementation Required|
| --- | --- |
|preemptdisabled|None|
|preemptenabled|Must implementKernelGuardIftrait|

### Runtime Interface Resolution

The crate uses `crate_interface::call_interface!` macros to dynamically dispatch to user-provided implementations at runtime. This occurs only when the `preempt` feature is active:

```mermaid
flowchart TD
FEATURE_ENABLED["preempt featureenabled"]
MACRO_CALL["crate_interface::call_interface!(KernelGuardIf::disable_preempt)"]
RUNTIME_DISPATCH["Runtime InterfaceResolution"]
USER_IMPL["User Implementationof KernelGuardIf"]
ACTUAL_DISABLE["Actual PreemptionDisable Operation"]
FEATURE_DISABLED["preempt featuredisabled"]
COMPILE_SKIP["Conditional CompilationSkips Call"]
NO_OPERATION["No PreemptionControl"]

COMPILE_SKIP --> NO_OPERATION
FEATURE_DISABLED --> COMPILE_SKIP
FEATURE_ENABLED --> MACRO_CALL
MACRO_CALL --> RUNTIME_DISPATCH
RUNTIME_DISPATCH --> USER_IMPL
USER_IMPL --> ACTUAL_DISABLE
```

**Sources**: [src/lib.rs(L153 - L154)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/lib.rs#L153-L154) [src/lib.rs(L158 - L159)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/lib.rs#L158-L159) [src/lib.rs(L167 - L168)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/lib.rs#L167-L168) [src/lib.rs(L176 - L177)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/lib.rs#L176-L177)

## Configuration Examples

### Basic Usage Without Preemption

```
[dependencies]
kernel_guard = "0.1"
```

In this configuration, `NoPreempt` and `NoPreemptIrqSave` guards will only control IRQs, making them functionally equivalent to `IrqSave` for the IRQ portion.

### Full Preemptive Configuration

```
[dependencies]
kernel_guard = { version = "0.1", features = ["preempt"] }
```

This configuration enables full preemption control capabilities, requiring the user to implement `KernelGuardIf`.

**Sources**: [Cargo.toml(L14 - L16)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/Cargo.toml#L14-L16) [src/lib.rs(L21 - L26)&emsp;](https://github.com/arceos-org/kernel_guard/blob/f1a9da26/src/lib.rs#L21-L26)