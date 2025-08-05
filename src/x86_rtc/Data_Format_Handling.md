# Data Format Handling

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs)

This document covers the data format conversion and handling mechanisms within the x86_rtc crate. The RTC driver must handle multiple data representation formats used by CMOS hardware, including BCD/binary encoding, 12/24-hour time formats, and Unix timestamp conversions. This section focuses specifically on the format detection, conversion algorithms, and data consistency mechanisms.

For hardware register access details, see [CMOS Hardware Interface](/arceos-org/x86_rtc/2.2-cmos-hardware-interface). For the high-level API usage, see [RTC Driver API](/arceos-org/x86_rtc/2.1-rtc-driver-api).

## Format Detection and Configuration

The RTC hardware can store time values in different formats, and the driver must detect and handle these variations dynamically. The format configuration is stored in the CMOS Status Register B and cached in the `Rtc` struct.

```mermaid
flowchart TD
INIT["Rtc::new()"]
READ_STATUS["read_cmos_register(CMOS_STATUS_REGISTER_B)"]
CACHE["cmos_format: u8"]
CHECK_24H["is_24_hour_format()"]
CHECK_BIN["is_binary_format()"]
FLAG_24H["CMOS_24_HOUR_FORMAT_FLAG"]
FLAG_BIN["CMOS_BINARY_FORMAT_FLAG"]
DECISION_TIME["Time Format Decision"]
DECISION_DATA["Data Format Decision"]
FORMAT_12H["12-Hour Format"]
FORMAT_24H["24-Hour Format"]
FORMAT_BCD["BCD Format"]
FORMAT_BINARY["Binary Format"]

CACHE --> CHECK_24H
CACHE --> CHECK_BIN
CHECK_24H --> FLAG_24H
CHECK_BIN --> FLAG_BIN
DECISION_DATA --> FORMAT_BCD
DECISION_DATA --> FORMAT_BINARY
DECISION_TIME --> FORMAT_12H
DECISION_TIME --> FORMAT_24H
FLAG_24H --> DECISION_TIME
FLAG_BIN --> DECISION_DATA
INIT --> READ_STATUS
READ_STATUS --> CACHE
```

The format detection methods use bitwise operations to check specific flags:

|Method|Flag Constant|Bit Position|Purpose|
| --- | --- | --- | --- |
|is_24_hour_format()|CMOS_24_HOUR_FORMAT_FLAG|Bit 1|Determines 12/24 hour format|
|is_binary_format()|CMOS_BINARY_FORMAT_FLAG|Bit 2|Determines BCD/binary encoding|

Sources: [src/lib.rs(L30 - L36)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L30-L36) [src/lib.rs(L20 - L21)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L20-L21) [src/lib.rs(L97 - L101)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L97-L101)

## BCD and Binary Format Conversion

The CMOS chip can store datetime values in either Binary-Coded Decimal (BCD) or pure binary format. The driver provides bidirectional conversion functions to handle both representations.

```mermaid
flowchart TD
subgraph subGraph0["BCD to Binary Conversion"]
    subgraph subGraph1["Binary to BCD Conversion"]
        BIN_INPUT["Binary Value (23)"]
        BIN_FUNC["convert_binary_value()"]
        BIN_TENS["Tens: binary / 10"]
        BIN_ONES["Ones: binary % 10"]
        BIN_CALC["((binary / 10) << 4) | (binary % 10)"]
        BIN_OUTPUT["BCD Value (0x23)"]
        BCD_INPUT["BCD Value (0x23)"]
        BCD_FUNC["convert_bcd_value()"]
        BCD_EXTRACT["Extract Tens: (0xF0) >> 4"]
        BCD_ONES["Extract Ones: (0x0F)"]
        BCD_CALC["((bcd & 0xF0) >> 1) + ((bcd & 0xF0) >> 3) + (bcd & 0x0f)"]
        BCD_OUTPUT["Binary Value (23)"]
    end
end

BCD_CALC --> BCD_OUTPUT
BCD_EXTRACT --> BCD_CALC
BCD_FUNC --> BCD_EXTRACT
BCD_FUNC --> BCD_ONES
BCD_INPUT --> BCD_FUNC
BCD_ONES --> BCD_CALC
BIN_CALC --> BIN_OUTPUT
BIN_FUNC --> BIN_ONES
BIN_FUNC --> BIN_TENS
BIN_INPUT --> BIN_FUNC
BIN_ONES --> BIN_CALC
BIN_TENS --> BIN_CALC
```

The `read_datetime_register()` method automatically applies the appropriate conversion based on the detected format:

```mermaid
flowchart TD
READ_REG["read_datetime_register(register)"]
READ_CMOS["read_cmos_register(register)"]
CHECK_FORMAT["is_binary_format()"]
RETURN_DIRECT["Return value directly"]
CONVERT_BCD["convert_bcd_value(value)"]
OUTPUT["Final Value"]

CHECK_FORMAT --> CONVERT_BCD
CHECK_FORMAT --> RETURN_DIRECT
CONVERT_BCD --> OUTPUT
READ_CMOS --> CHECK_FORMAT
READ_REG --> READ_CMOS
RETURN_DIRECT --> OUTPUT
```

Sources: [src/lib.rs(L38 - L48)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L38-L48) [src/lib.rs(L253 - L260)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L253-L260) [src/lib.rs(L171 - L177)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L171-L177)

## Hour Format Handling

Hour values require the most complex format handling due to the combination of BCD/binary encoding with 12/24-hour format variations and PM flag management.

```mermaid
flowchart TD
READ_HOUR["read_cmos_register(CMOS_HOUR_REGISTER)"]
CHECK_12H["is_24_hour_format()"]
EXTRACT_PM["time_is_pm(hour)"]
SKIP_PM["Skip PM handling"]
MASK_PM["hour &= !CMOS_12_HOUR_PM_FLAG"]
CHECK_BCD_12H["is_binary_format()"]
CHECK_BCD_24H["is_binary_format()"]
CONVERT_BCD_12H["convert_bcd_value(hour)"]
HOUR_READY_12H["hour ready for 12h conversion"]
CONVERT_BCD_24H["convert_bcd_value(hour)"]
HOUR_FINAL["Final 24h hour"]
CONVERT_12_TO_24["Convert 12h to 24h"]
CHECK_NOON["hour == 12"]
SET_ZERO["hour = 0"]
KEEP_HOUR["Keep hour value"]
CHECK_PM_FINAL["is_pm"]
ADD_12["hour += 12"]

CHECK_12H --> EXTRACT_PM
CHECK_12H --> SKIP_PM
CHECK_BCD_12H --> CONVERT_BCD_12H
CHECK_BCD_12H --> HOUR_READY_12H
CHECK_BCD_24H --> CONVERT_BCD_24H
CHECK_BCD_24H --> HOUR_FINAL
CHECK_NOON --> KEEP_HOUR
CHECK_NOON --> SET_ZERO
CHECK_PM_FINAL --> ADD_12
CHECK_PM_FINAL --> HOUR_FINAL
CONVERT_BCD_12H --> CONVERT_12_TO_24
CONVERT_BCD_24H --> HOUR_FINAL
EXTRACT_PM --> MASK_PM
HOUR_READY_12H --> CONVERT_12_TO_24
KEEP_HOUR --> CHECK_PM_FINAL
MASK_PM --> CHECK_BCD_12H
READ_HOUR --> CHECK_12H
SET_ZERO --> CHECK_PM_FINAL
SKIP_PM --> CHECK_BCD_24H
```

The conversion logic handles these specific cases:

|Input (12h)|is_pm|Converted (24h)|Logic|
| --- | --- | --- | --- |
|12:xx AM|false|00:xx|hour = 0|
|01:xx AM|false|01:xx|hour unchanged|
|12:xx PM|true|12:xx|hour = 0 + 12 = 12|
|01:xx PM|true|13:xx|hour = 1 + 12 = 13|

Sources: [src/lib.rs(L56 - L84)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L56-L84) [src/lib.rs(L247 - L249)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L247-L249) [src/lib.rs(L22)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L22-L22)

## Unix Timestamp Conversion

The driver provides bidirectional conversion between CMOS datetime values and Unix timestamps (seconds since January 1, 1970).

### Reading: CMOS to Unix Timestamp

```mermaid
flowchart TD
GET_TIMESTAMP["get_unix_timestamp()"]
WAIT_STABLE["Wait for !UPDATE_IN_PROGRESS"]
READ_ALL["read_all_values()"]
READ_YEAR["read_datetime_register(CMOS_YEAR_REGISTER) + 2000"]
READ_MONTH["read_datetime_register(CMOS_MONTH_REGISTER)"]
READ_DAY["read_datetime_register(CMOS_DAY_REGISTER)"]
PROCESS_HOUR["Complex hour processing"]
READ_MINUTE["read_datetime_register(CMOS_MINUTE_REGISTER)"]
READ_SECOND["read_datetime_register(CMOS_SECOND_REGISTER)"]
CALC_UNIX["seconds_from_date()"]
CHECK_CONSISTENT["Verify consistency"]
RETURN_TIMESTAMP["Return Unix timestamp"]

CALC_UNIX --> CHECK_CONSISTENT
CHECK_CONSISTENT --> RETURN_TIMESTAMP
CHECK_CONSISTENT --> WAIT_STABLE
GET_TIMESTAMP --> WAIT_STABLE
PROCESS_HOUR --> CALC_UNIX
READ_ALL --> PROCESS_HOUR
READ_ALL --> READ_DAY
READ_ALL --> READ_MINUTE
READ_ALL --> READ_MONTH
READ_ALL --> READ_SECOND
READ_ALL --> READ_YEAR
READ_DAY --> CALC_UNIX
READ_MINUTE --> CALC_UNIX
READ_MONTH --> CALC_UNIX
READ_SECOND --> CALC_UNIX
READ_YEAR --> CALC_UNIX
WAIT_STABLE --> READ_ALL
```

### Writing: Unix Timestamp to CMOS

```mermaid
flowchart TD
SET_TIMESTAMP["set_unix_timestamp(unix_time)"]
CALC_TIME["Calculate hour, minute, second"]
CALC_DATE["Calculate year, month, day"]
EXTRACT_HOUR["hour = t / 3600"]
EXTRACT_MIN["minute = (t % 3600) / 60"]
EXTRACT_SEC["second = t % 60"]
YEAR_LOOP["Subtract full years since 1970"]
MONTH_LOOP["Subtract full months"]
CALC_DAY["Remaining days + 1"]
CHECK_BIN_FORMAT["is_binary_format()"]
CONVERT_TO_BCD["convert_binary_value() for each"]
WRITE_REGISTERS["Write to CMOS registers"]
WRITE_SEC["write_cmos_register(CMOS_SECOND_REGISTER)"]
WRITE_MIN["write_cmos_register(CMOS_MINUTE_REGISTER)"]
WRITE_HOUR["write_cmos_register(CMOS_HOUR_REGISTER)"]
WRITE_DAY["write_cmos_register(CMOS_DAY_REGISTER)"]
WRITE_MONTH["write_cmos_register(CMOS_MONTH_REGISTER)"]
WRITE_YEAR["write_cmos_register(CMOS_YEAR_REGISTER)"]

CALC_DATE --> YEAR_LOOP
CALC_DAY --> CHECK_BIN_FORMAT
CALC_TIME --> EXTRACT_HOUR
CALC_TIME --> EXTRACT_MIN
CALC_TIME --> EXTRACT_SEC
CHECK_BIN_FORMAT --> CONVERT_TO_BCD
CHECK_BIN_FORMAT --> WRITE_REGISTERS
CONVERT_TO_BCD --> WRITE_REGISTERS
EXTRACT_HOUR --> CHECK_BIN_FORMAT
EXTRACT_MIN --> CHECK_BIN_FORMAT
EXTRACT_SEC --> CHECK_BIN_FORMAT
MONTH_LOOP --> CALC_DAY
SET_TIMESTAMP --> CALC_DATE
SET_TIMESTAMP --> CALC_TIME
WRITE_REGISTERS --> WRITE_DAY
WRITE_REGISTERS --> WRITE_HOUR
WRITE_REGISTERS --> WRITE_MIN
WRITE_REGISTERS --> WRITE_MONTH
WRITE_REGISTERS --> WRITE_SEC
WRITE_REGISTERS --> WRITE_YEAR
YEAR_LOOP --> MONTH_LOOP
```

Sources: [src/lib.rs(L106 - L129)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L106-L129) [src/lib.rs(L132 - L193)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L132-L193) [src/lib.rs(L264 - L276)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L264-L276)

## Data Consistency and Synchronization

The CMOS hardware updates its registers periodically, which can cause inconsistent reads if accessed during an update cycle. The driver implements a consistency check mechanism.

```mermaid
flowchart TD
START_READ["Start get_unix_timestamp()"]
CHECK_UPDATE1["Read CMOS_STATUS_REGISTER_A"]
UPDATE_FLAG1["Check CMOS_UPDATE_IN_PROGRESS_FLAG"]
SPIN_WAIT["core::hint::spin_loop()"]
READ_VALUES1["Read all datetime values"]
CHECK_UPDATE2["Check update flag again"]
READ_VALUES2["Read all datetime values again"]
COMPARE["Compare both readings"]
RETURN_CONSISTENT["Return consistent value"]

CHECK_UPDATE1 --> UPDATE_FLAG1
CHECK_UPDATE2 --> READ_VALUES2
CHECK_UPDATE2 --> START_READ
COMPARE --> RETURN_CONSISTENT
COMPARE --> START_READ
READ_VALUES1 --> CHECK_UPDATE2
READ_VALUES2 --> COMPARE
SPIN_WAIT --> CHECK_UPDATE1
START_READ --> CHECK_UPDATE1
UPDATE_FLAG1 --> READ_VALUES1
UPDATE_FLAG1 --> SPIN_WAIT
```

This double-read mechanism ensures that:

1. No update is in progress before reading
2. No update occurred during reading
3. Two consecutive reads produce identical results

Sources: [src/lib.rs(L107 - L129)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L107-L129) [src/lib.rs(L19)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L19-L19)

## Calendar Arithmetic Functions

The driver includes helper functions for calendar calculations needed during timestamp conversion:

|Function|Purpose|Key Logic|
| --- | --- | --- |
|is_leap_year()|Leap year detection|(year % 4 == 0 && year % 100 != 0) \|\| (year % 400 == 0)|
|days_in_month()|Days per month|Handles leap year February (28/29 days)|
|seconds_from_date()|Date to Unix timestamp|Uses optimized algorithm from Linux kernel|

The `seconds_from_date()` function uses an optimized algorithm that adjusts the year and month to simplify leap year calculations by treating March as the start of the year.

Sources: [src/lib.rs(L228 - L245)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L228-L245) [src/lib.rs(L264 - L276)&emsp;](https://github.com/arceos-org/x86_rtc/blob/1990537d/src/lib.rs#L264-L276)