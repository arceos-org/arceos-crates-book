# Memory Management Internals

> **Relevant source files**
> * [percpu/build.rs](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/build.rs)
> * [percpu/src/imp.rs](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs)
> * [percpu/test_percpu.x](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/test_percpu.x)

This document covers the low-level memory management implementation within the percpu crate, focusing on per-CPU data area allocation, address calculation, and register management. This section details the internal mechanisms that support the per-CPU data abstraction layer.

For high-level memory layout concepts, see [Architecture and Design](/arceos-org/percpu/3-architecture-and-design). For architecture-specific code generation details, see [Architecture-Specific Code Generation](/arceos-org/percpu/5.1-architecture-specific-code-generation). For the single-CPU fallback implementation, see [Naive Implementation](/arceos-org/percpu/5.2-naive-implementation).

## Memory Area Lifecycle

The per-CPU memory management system follows a well-defined lifecycle from initialization through runtime access. The process begins with linker script integration and proceeds through dynamic allocation and template copying.

### Initialization Process

The initialization process is controlled by the `init()` function which ensures single initialization and handles platform-specific memory allocation:

```mermaid
flowchart TD
START["init()"]
CHECK["IS_INIT.compare_exchange()"]
RETURN_ZERO["return 0"]
PLATFORM_CHECK["Platform Check"]
ALLOC["std::alloc::alloc()"]
USE_LINKER["Use _percpu_start"]
SET_BASE["PERCPU_AREA_BASE.call_once()"]
COPY_LOOP["Copy Template Loop"]
COPY_PRIMARY["copy_nonoverlapping(base, secondary_base, size)"]
RETURN_NUM["return num"]
END["END"]

ALLOC --> SET_BASE
CHECK --> PLATFORM_CHECK
CHECK --> RETURN_ZERO
COPY_LOOP --> COPY_PRIMARY
COPY_PRIMARY --> COPY_LOOP
COPY_PRIMARY --> RETURN_NUM
PLATFORM_CHECK --> ALLOC
PLATFORM_CHECK --> USE_LINKER
RETURN_NUM --> END
RETURN_ZERO --> END
SET_BASE --> COPY_LOOP
START --> CHECK
USE_LINKER --> COPY_LOOP
```

**Memory Area Initialization Sequence**
Sources: [percpu/src/imp.rs(L56 - L86)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L56-L86)

The `IS_INIT` atomic boolean prevents re-initialization using compare-and-swap semantics. On Linux platforms, the system dynamically allocates memory since the `.percpu` section is not loaded in ELF files. On bare metal (`target_os = "none"`), the linker-defined symbols provide the memory area directly.

### Template Copying Mechanism

Per-CPU areas are initialized by copying from a primary template area to secondary CPU areas:

```mermaid
flowchart TD
TEMPLATE["Primary CPU Areabase = percpu_area_base(0)"]
CPU1["CPU 1 Areapercpu_area_base(1)"]
CPU2["CPU 2 Areapercpu_area_base(2)"]
CPUN["CPU N Areapercpu_area_base(N)"]
COPY1["copy_nonoverlapping(base, base+offset1, size)"]
COPY2["copy_nonoverlapping(base, base+offset2, size)"]
COPYN["copy_nonoverlapping(base, base+offsetN, size)"]

CPU1 --> COPY1
CPU2 --> COPY2
CPUN --> COPYN
TEMPLATE --> CPU1
TEMPLATE --> CPU2
TEMPLATE --> CPUN
```

**Template to Per-CPU Area Copying**
Sources: [percpu/src/imp.rs(L76 - L84)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L76-L84)

## Address Calculation and Layout

The memory layout uses 64-byte aligned areas calculated through several key functions that work together to provide consistent addressing across different platforms.

### Address Calculation Functions

```mermaid
flowchart TD
subgraph subGraph0["Platform Specific"]
    LINUX_BASE["PERCPU_AREA_BASE.get()"]
    BAREMETAL_BASE["_percpu_start as usize"]
end
START_SYM["_percpu_start"]
AREA_BASE["percpu_area_base(cpu_id)"]
SIZE_CALC["percpu_area_size()"]
ALIGN["align_up_64()"]
FINAL_ADDR["Final Address"]
LOAD_START["_percpu_load_start"]
LOAD_END["_percpu_load_end"]
OFFSET["percpu_symbol_offset!()"]
NUM_CALC["percpu_area_num()"]
END_SYM["_percpu_end"]

ALIGN --> AREA_BASE
AREA_BASE --> FINAL_ADDR
AREA_BASE --> NUM_CALC
BAREMETAL_BASE --> AREA_BASE
END_SYM --> NUM_CALC
LINUX_BASE --> AREA_BASE
LOAD_END --> SIZE_CALC
LOAD_START --> SIZE_CALC
SIZE_CALC --> ALIGN
SIZE_CALC --> OFFSET
START_SYM --> AREA_BASE
```

**Address Calculation Dependencies**
Sources: [percpu/src/imp.rs(L21 - L44)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L21-L44)

The `align_up_64()` function ensures all per-CPU areas are aligned to 64-byte boundaries for cache line optimization:

|Function|Purpose|Return Value|
| --- | --- | --- |
|percpu_area_size()|Template area size|Size in bytes from linker symbols|
|percpu_area_num()|Number of CPU areas|Total section size / aligned area size|
|percpu_area_base(cpu_id)|CPU-specific base address|Base + (cpu_id * aligned_size)|
|align_up_64(val)|64-byte alignment|(val + 63) & !63|

Sources: [percpu/src/imp.rs(L5 - L8)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L5-L8) [percpu/src/imp.rs(L25 - L30)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L25-L30) [percpu/src/imp.rs(L32 - L44)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L32-L44) [percpu/src/imp.rs(L20 - L23)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L20-L23)

## Register Management Internals

Each CPU architecture uses a dedicated register to hold the per-CPU data base address. The register management system provides unified access across different instruction sets.

### Architecture-Specific Register Access

```mermaid
flowchart TD
READ_REG["read_percpu_reg()"]
ARCH_DETECT["Architecture Detection"]
WRITE_REG["write_percpu_reg(tp)"]
X86["target_arch = x86_64"]
ARM64["target_arch = aarch64"]
RISCV["target_arch = riscv32/64"]
LOONG["target_arch = loongarch64"]
X86_READ["rdmsr(IA32_GS_BASE)or SELF_PTR.read_current_raw()"]
X86_WRITE["wrmsr(IA32_GS_BASE, tp)or arch_prctl(ARCH_SET_GS, tp)"]
ARM_READ["mrs TPIDR_EL1/EL2"]
ARM_WRITE["msr TPIDR_EL1/EL2, tp"]
RISCV_READ["mv tp, gp"]
RISCV_WRITE["mv gp, tp"]
LOONG_READ["move tp, $r21"]
LOONG_WRITE["move $r21, tp"]

ARCH_DETECT --> ARM64
ARCH_DETECT --> LOONG
ARCH_DETECT --> RISCV
ARCH_DETECT --> X86
ARM64 --> ARM_READ
ARM64 --> ARM_WRITE
LOONG --> LOONG_READ
LOONG --> LOONG_WRITE
READ_REG --> ARCH_DETECT
RISCV --> RISCV_READ
RISCV --> RISCV_WRITE
WRITE_REG --> ARCH_DETECT
X86 --> X86_READ
X86 --> X86_WRITE
```

**Per-CPU Register Access by Architecture**
Sources: [percpu/src/imp.rs(L88 - L117)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L88-L117) [percpu/src/imp.rs(L119 - L156)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L119-L156)

### Register Initialization Process

The `init_percpu_reg(cpu_id)` function combines address calculation with register writing:

```mermaid
flowchart TD
INIT_CALL["init_percpu_reg(cpu_id)"]
CALC_BASE["percpu_area_base(cpu_id)"]
GET_TP["tp = calculated_address"]
WRITE_REG["write_percpu_reg(tp)"]
REG_SET["CPU register updated"]

CALC_BASE --> GET_TP
GET_TP --> WRITE_REG
INIT_CALL --> CALC_BASE
WRITE_REG --> REG_SET
```

**Register Initialization Flow**
Sources: [percpu/src/imp.rs(L158 - L168)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L158-L168)

The x86_64 architecture requires special handling with the `SELF_PTR` variable that stores the per-CPU base address within the per-CPU area itself:

```mermaid
flowchart TD
X86_SPECIAL["x86_64 Special Case"]
SELF_PTR_DEF["SELF_PTR: usize = 0"]
PERCPU_MACRO["#[def_percpu] static"]
GS_ACCESS["gs:SELF_PTR access"]
CIRCULAR["Circular reference for self-location"]

GS_ACCESS --> CIRCULAR
PERCPU_MACRO --> GS_ACCESS
SELF_PTR_DEF --> PERCPU_MACRO
X86_SPECIAL --> SELF_PTR_DEF
```

**x86_64 Self-Pointer Mechanism**
Sources: [percpu/src/imp.rs(L174 - L178)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L174-L178)

## Linker Integration Details

The linker script integration creates the necessary memory layout for per-CPU data areas through carefully designed section definitions.

### Linker Script Structure

```mermaid
flowchart TD
LINKER_START[".percpu section definition"]
SYMBOLS["Symbol Generation"]
START_SYM["_percpu_start = ."]
END_SYM["_percpu_end = _percpu_start + SIZEOF(.percpu)"]
SECTION_DEF[".percpu 0x0 (NOLOAD) : AT(_percpu_start)"]
LOAD_START["_percpu_load_start = ."]
CONTENT["(.percpu .percpu.)"]
LOAD_END["_percpu_load_end = ."]
ALIGN_EXPAND[". = _percpu_load_start + ALIGN(64) * CPU_NUM"]
VARS["Per-CPU Variables"]
SPACE_RESERVE["Reserved space for all CPUs"]

ALIGN_EXPAND --> SPACE_RESERVE
CONTENT --> VARS
LINKER_START --> SECTION_DEF
LINKER_START --> SYMBOLS
SECTION_DEF --> ALIGN_EXPAND
SECTION_DEF --> CONTENT
SECTION_DEF --> LOAD_END
SECTION_DEF --> LOAD_START
SYMBOLS --> END_SYM
SYMBOLS --> START_SYM
```

**Linker Script Memory Layout**
Sources: [percpu/test_percpu.x(L1 - L16)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/test_percpu.x#L1-L16)

|Symbol|Purpose|Usage|
| --- | --- | --- |
|_percpu_start|Section start address|Base address calculation|
|_percpu_end|Section end address|Total size calculation|
|_percpu_load_start|Template start|Size calculation for copying|
|_percpu_load_end|Template end|Size calculation for copying|

The `NOLOAD` directive ensures the section occupies space but isn't loaded from the ELF file, while `AT(_percpu_start)` specifies the load address for the template data.

### Build System Integration

The build system conditionally applies linker scripts based on target platform:

```mermaid
flowchart TD
BUILD_CHECK["build.rs"]
LINUX_CHECK["cfg!(target_os = linux)"]
FEATURE_CHECK["cfg!(not(feature = sp-naive))"]
APPLY_SCRIPT["rustc-link-arg-tests=-T test_percpu.x"]
NO_PIE["rustc-link-arg-tests=-no-pie"]

BUILD_CHECK --> LINUX_CHECK
FEATURE_CHECK --> APPLY_SCRIPT
FEATURE_CHECK --> NO_PIE
LINUX_CHECK --> FEATURE_CHECK
```

**Build System Linker Integration**
Sources: [percpu/build.rs(L3 - L9)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/build.rs#L3-L9)

## Platform-Specific Considerations

Different target platforms require distinct memory management strategies due to varying levels of hardware access and memory management capabilities.

### Platform Memory Allocation Matrix

|Platform|Allocation Method|Base Address Source|Register Access|
| --- | --- | --- | --- |
|Linux userspace|std::alloc::alloc()|PERCPU_AREA_BASE|arch_prctl()syscall|
|Bare metal|Linker symbols|_percpu_start|Direct MSR/register|
|Kernel/Hypervisor|Linker symbols|_percpu_start|Privileged instructions|

The `PERCPU_AREA_BASE` is a `spin::once::Once<usize>` that ensures thread-safe initialization of the dynamically allocated base address on platforms where the linker section is not available.

### Error Handling and Validation

The system includes several validation mechanisms:

* Atomic initialization checking with `IS_INIT`
* Address range validation on bare metal platforms
* Alignment verification through `align_up_64()`
* Platform capability detection through conditional compilation

Sources: [percpu/src/imp.rs(L1 - L179)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/src/imp.rs#L1-L179) [percpu/test_percpu.x(L1 - L16)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/test_percpu.x#L1-L16) [percpu/build.rs(L1 - L9)&emsp;](https://github.com/arceos-org/percpu/blob/89c8a54c/percpu/build.rs#L1-L9)