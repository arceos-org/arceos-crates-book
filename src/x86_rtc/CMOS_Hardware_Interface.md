# CMOS Hardware Interface

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs)

This document explains the low-level hardware interface for accessing the Real Time Clock (RTC) through CMOS registers on x86_64 systems. It covers the I/O port protocol, register mapping, hardware synchronization, and platform-specific implementation details.

For high-level RTC API usage, see [RTC Driver API](/arceos-org/x86_rtc/2.1-rtc-driver-api). For data format conversion details, see [Data Format Handling](/arceos-org/x86_rtc/2.3-data-format-handling).

## CMOS Register Map

The CMOS RTC uses a well-defined register layout accessible through I/O ports. The implementation defines specific register addresses and control flags for accessing time, date, and status information.

### Time and Date Registers

```mermaid
flowchart TD
subgraph subGraph3["Format Flags"]
    UIP["CMOS_UPDATE_IN_PROGRESS_FLAGBit 7"]
    H24["CMOS_24_HOUR_FORMAT_FLAGBit 1"]
    BIN["CMOS_BINARY_FORMAT_FLAGBit 2"]
    PM["CMOS_12_HOUR_PM_FLAG0x80"]
end
subgraph subGraph2["Status Registers"]
    DAY["CMOS_DAY_REGISTER0x07"]
    MONTH["CMOS_MONTH_REGISTER0x08"]
    YEAR["CMOS_YEAR_REGISTER0x09"]
    STATA["CMOS_STATUS_REGISTER_A0x0AUpdate Progress"]
    STATB["CMOS_STATUS_REGISTER_B0x0BFormat Control"]
end
subgraph subGraph0["Time Registers"]
    SEC["CMOS_SECOND_REGISTER0x00"]
    MIN["CMOS_MINUTE_REGISTER0x02"]
    HOUR["CMOS_HOUR_REGISTER0x04"]
end
subgraph subGraph1["Date Registers"]
    DAY["CMOS_DAY_REGISTER0x07"]
    MONTH["CMOS_MONTH_REGISTER0x08"]
    YEAR["CMOS_YEAR_REGISTER0x09"]
    STATA["CMOS_STATUS_REGISTER_A0x0AUpdate Progress"]
end

HOUR --> PM
STATA --> UIP
STATB --> BIN
STATB --> H24
```

**Sources:** [src/lib.rs(L10 - L23)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L10-L23)

### Register Access Pattern

|Register|Address|Purpose|Format Dependencies|
| --- | --- | --- | --- |
|CMOS_SECOND_REGISTER|0x00|Current second (0-59)|BCD/Binary|
|CMOS_MINUTE_REGISTER|0x02|Current minute (0-59)|BCD/Binary|
|CMOS_HOUR_REGISTER|0x04|Current hour|BCD/Binary + 12/24-hour|
|CMOS_DAY_REGISTER|0x07|Day of month (1-31)|BCD/Binary|
|CMOS_MONTH_REGISTER|0x08|Month (1-12)|BCD/Binary|
|CMOS_YEAR_REGISTER|0x09|Year (0-99, + 2000)|BCD/Binary|
|CMOS_STATUS_REGISTER_A|0x0A|Update status|Raw binary|
|CMOS_STATUS_REGISTER_B|0x0B|Format configuration|Raw binary|

## I/O Port Protocol

The CMOS interface uses a two-port protocol where register selection and data transfer are performed through separate I/O ports.

### Port Configuration

```mermaid
flowchart TD
subgraph subGraph2["Access Functions"]
    READ["read_cmos_register()"]
    WRITE["write_cmos_register()"]
end
subgraph subGraph1["Port Operations"]
    CMDPORT["Port<u8>COMMAND_PORT"]
    DATAPORT["Port<u8>DATA_PORT"]
end
subgraph subGraph0["I/O Port Interface"]
    CMD["CMOS_COMMAND_PORT0x70Register Selection"]
    DATA["CMOS_DATA_PORT0x71Data Transfer"]
end

CMD --> CMDPORT
CMDPORT --> READ
CMDPORT --> WRITE
DATA --> DATAPORT
DATAPORT --> READ
DATAPORT --> WRITE
```

**Sources:** [src/lib.rs(L198 - L204)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L198-L204)

### Hardware Communication Protocol

The CMOS access protocol follows a strict sequence to ensure reliable register access:

```mermaid
sequenceDiagram
    participant CPU as CPU
    participant COMMAND_PORT0x70 as "COMMAND_PORT (0x70)"
    participant DATA_PORT0x71 as "DATA_PORT (0x71)"
    participant CMOSChip as "CMOS Chip"

    Note over CPU: Read Operation
    CPU ->> COMMAND_PORT0x70: "CMOS_DISABLE_NMI | register"
    COMMAND_PORT0x70 ->> CMOSChip: "Select Register"
    CPU ->> DATA_PORT0x71: "read()"
    DATA_PORT0x71 ->> CMOSChip: "Read Request"
    CMOSChip ->> DATA_PORT0x71: "Register Value"
    DATA_PORT0x71 ->> CPU: "Return Value"
    Note over CPU: Write Operation
    CPU ->> COMMAND_PORT0x70: "CMOS_DISABLE_NMI | register"
    COMMAND_PORT0x70 ->> CMOSChip: "Select Register"
    CPU ->> DATA_PORT0x71: "write(value)"
    DATA_PORT0x71 ->> CMOSChip: "Write Value"
```

**Sources:** [src/lib.rs(L206 - L218)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L206-L218)

## Hardware Access Implementation

### Register Read Operation

The `read_cmos_register` function implements the low-level hardware access protocol:

```mermaid
flowchart TD
subgraph subGraph1["Hardware Control"]
    NMI["CMOS_DISABLE_NMIBit 7 = 1Prevents interrupts"]
    REGSEL["Register SelectionBits 0-6"]
end
subgraph subGraph0["read_cmos_register Function"]
    START["Start: register parameter"]
    SELECTREG["Write to COMMAND_PORTCMOS_DISABLE_NMI | register"]
    READDATA["Read from DATA_PORT"]
    RETURN["Return u8 value"]
end

READDATA --> RETURN
SELECTREG --> NMI
SELECTREG --> READDATA
SELECTREG --> REGSEL
START --> SELECTREG
```

**Sources:** [src/lib.rs(L206 - L211)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L206-L211)

### Register Write Operation

The `write_cmos_register` function handles data updates to CMOS registers:

```mermaid
flowchart TD
subgraph subGraph1["Safety Considerations"]
    UNSAFE["unsafe blockRaw port access"]
    RAWPTR["raw mut pointerStatic port references"]
end
subgraph subGraph0["write_cmos_register Function"]
    START["Start: register, value parameters"]
    SELECTREG["Write to COMMAND_PORTCMOS_DISABLE_NMI | register"]
    WRITEDATA["Write value to DATA_PORT"]
    END["End"]
end

SELECTREG --> UNSAFE
SELECTREG --> WRITEDATA
START --> SELECTREG
WRITEDATA --> END
WRITEDATA --> RAWPTR
```

**Sources:** [src/lib.rs(L213 - L218)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L213-L218)

## Status Register Management

### Update Synchronization

The CMOS chip updates its registers autonomously, requiring careful synchronization to avoid reading inconsistent values:

```mermaid
flowchart TD
subgraph subGraph0["Update Detection Flow"]
    CHECK1["Read CMOS_STATUS_REGISTER_A"]
    TESTFLAG1["Test CMOS_UPDATE_IN_PROGRESS_FLAG"]
    WAIT["spin_loop() while updating"]
    READ1["Read all time registers"]
    CHECK2["Read CMOS_STATUS_REGISTER_A again"]
    TESTFLAG2["Test update flag again"]
    READ2["Read all time registers again"]
    COMPARE["Compare both readings"]
    RETURN["Return consistent value"]
end
read2["read2"]

CHECK1 --> TESTFLAG1
CHECK2 --> TESTFLAG2
COMPARE --> CHECK1
COMPARE --> RETURN
READ1 --> CHECK2
TESTFLAG1 --> READ1
TESTFLAG1 --> WAIT
TESTFLAG2 --> CHECK1
TESTFLAG2 --> READ2
WAIT --> CHECK1
read2 --> COMPARE
```

**Sources:** [src/lib.rs(L107 - L129)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L107-L129)

### Format Detection

The implementation reads `CMOS_STATUS_REGISTER_B` during initialization to determine data format:

|Flag|Bit Position|Purpose|Impact|
| --- | --- | --- | --- |
|CMOS_24_HOUR_FORMAT_FLAG|1|Hour format detection|Affects hour register interpretation|
|CMOS_BINARY_FORMAT_FLAG|2|Number format detection|Determines BCD vs binary conversion|

**Sources:** [src/lib.rs(L30 - L36)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L30-L36)

## Platform Abstraction

### Conditional Compilation

The implementation uses conditional compilation to provide platform-specific functionality:

```mermaid
flowchart TD
subgraph subGraph2["Fallback Implementation"]
    STUBREAD["Stub read_cmos_register()Returns 0"]
    STUBWRITE["Stub write_cmos_register()No operation"]
end
subgraph subGraph1["x86/x86_64 Implementation"]
    REALPORTS["Real I/O Port Accessx86_64::instructions::port::Port"]
    REALREAD["Actual CMOS register reads"]
    REALWRITE["Actual CMOS register writes"]
end
subgraph subGraph0["Compilation Targets"]
    X86["x86/x86_64target_arch"]
    OTHER["Other Architectures"]
end

OTHER --> STUBREAD
OTHER --> STUBWRITE
REALPORTS --> REALREAD
REALPORTS --> REALWRITE
X86 --> REALPORTS
```

**Sources:** [src/lib.rs(L196 - L226)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L196-L226)

### Safety Considerations

The hardware access requires `unsafe` code due to direct I/O port manipulation:

* **Raw pointer access**: Static `Port<u8>` instances require raw mutable references
* **Interrupt safety**: `CMOS_DISABLE_NMI` flag prevents hardware interrupts during access
* **Atomic operations**: The two-port protocol ensures atomic register selection and data transfer

**Sources:** [src/lib.rs(L207 - L217)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L207-L217)