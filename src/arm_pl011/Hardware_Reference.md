# Hardware Reference

> **Relevant source files**
> * [README.md](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/README.md)
> * [src/pl011.rs](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs)

This document provides technical specifications and hardware details for the ARM PL011 UART controller. It covers register definitions, memory mapping, signal interfaces, and hardware configuration requirements necessary for embedded systems development with the PL011.

For software implementation details and API usage, see [Core Implementation](/arceos-org/arm_pl011/2-core-implementation). For register abstraction and driver implementation, see [Register Definitions](/arceos-org/arm_pl011/2.1-register-definitions).

## PL011 UART Controller Overview

The ARM PL011 is a programmable serial interface controller that implements the Universal Asynchronous Receiver/Transmitter (UART) functionality. It provides full-duplex serial communication with configurable baud rates, data formats, and hardware flow control capabilities.

### Hardware Block Architecture

```mermaid
flowchart TD
subgraph System_Interface["System_Interface"]
    CLK["UARTCLK"]
    PCLK["PCLK"]
    RESET["PRESETn"]
    IRQ["UARTINTR"]
end
subgraph External_Signals["External_Signals"]
    TXD["TXD (Transmit Data)"]
    RXD["RXD (Receive Data)"]
    RTS["nRTS (Request to Send)"]
    CTS["nCTS (Clear to Send)"]
    DSR["nDSR (Data Set Ready)"]
    DTR["nDTR (Data Terminal Ready)"]
    DCD["nDCD (Data Carrier Detect)"]
    RI["nRI (Ring Indicator)"]
end
subgraph PL011_Hardware_Block["PL011_Hardware_Block"]
    APB["APB Interface"]
    CTRL["Control Logic"]
    TXFIFO["TX FIFO(16 entries)"]
    RXFIFO["RX FIFO(16 entries)"]
    BAUD["Baud Rate Generator"]
    SER["Serializer/Deserializer"]
    INT["Interrupt Controller"]
end

APB --> CTRL
BAUD --> SER
CLK --> BAUD
CTRL --> BAUD
CTRL --> INT
CTRL --> RXFIFO
CTRL --> TXFIFO
CTS --> SER
DCD --> SER
DSR --> SER
INT --> IRQ
PCLK --> APB
RESET --> CTRL
RI --> SER
RXD --> SER
SER --> DTR
SER --> RTS
SER --> RXFIFO
SER --> TXD
TXFIFO --> SER
```

Sources: [src/pl011.rs(L9 - L32)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L9-L32)

## Register Map and Memory Layout

The PL011 uses a memory-mapped register interface accessed through the APB bus. The register map spans 72 bytes (0x48) with specific registers at defined offsets.

### Complete Register Map

|Offset|Register|Access|Description|
| --- | --- | --- | --- |
|0x00|DR|R/W|Data Register|
|0x04-0x14|Reserved|-|Reserved space|
|0x18|FR|RO|Flag Register|
|0x1C-0x2C|Reserved|-|Reserved space|
|0x30|CR|R/W|Control Register|
|0x34|IFLS|R/W|Interrupt FIFO Level Select|
|0x38|IMSC|R/W|Interrupt Mask Set/Clear|
|0x3C|RIS|RO|Raw Interrupt Status|
|0x40|MIS|RO|Masked Interrupt Status|
|0x44|ICR|WO|Interrupt Clear|
|0x48|END|-|End of register space|

```mermaid
flowchart TD
subgraph Memory_Layout_0x48_bytes["Memory_Layout_0x48_bytes"]
    DR["0x00: DRData Register(R/W)"]
    RES1["0x04-0x14Reserved"]
    FR["0x18: FRFlag Register(RO)"]
    RES2["0x1C-0x2CReserved"]
    CR["0x30: CRControl Register(R/W)"]
    IFLS["0x34: IFLSInterrupt FIFO Level(R/W)"]
    IMSC["0x38: IMSCInterrupt Mask(R/W)"]
    RIS["0x3C: RISRaw Interrupt Status(RO)"]
    MIS["0x40: MISMasked Interrupt Status(RO)"]
    ICR["0x44: ICRInterrupt Clear(WO)"]
end

CR --> IFLS
DR --> RES1
FR --> RES2
IFLS --> IMSC
IMSC --> RIS
MIS --> ICR
RES1 --> FR
RES2 --> CR
RIS --> MIS
```

Sources: [src/pl011.rs(L9 - L32)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L9-L32)

## Register Specifications

### Data Register (DR) - Offset 0x00

The Data Register provides the interface for transmitting and receiving data. It handles both read and write operations with automatic FIFO management.

|Bits|Field|Access|Description|
| --- | --- | --- | --- |
|31:12|Reserved|-|Reserved, reads as zero|
|11|OE|RO|Overrun Error|
|10|BE|RO|Break Error|
|9|PE|RO|Parity Error|
|8|FE|RO|Framing Error|
|7:0|DATA|R/W|Transmit/Receive Data|

### Flag Register (FR) - Offset 0x18

The Flag Register provides status information about the UART's operational state and FIFO conditions.

|Bits|Field|Access|Description|
| --- | --- | --- | --- |
|31:8|Reserved|-|Reserved|
|7|TXFE|RO|Transmit FIFO Empty|
|6|RXFF|RO|Receive FIFO Full|
|5|TXFF|RO|Transmit FIFO Full|
|4|RXFE|RO|Receive FIFO Empty|
|3|BUSY|RO|UART Busy|
|2|DCD|RO|Data Carrier Detect|
|1|DSR|RO|Data Set Ready|
|0|CTS|RO|Clear to Send|

### Control Register (CR) - Offset 0x30

The Control Register configures the operational parameters of the UART including enables, loopback, and flow control.

|Bits|Field|Access|Description|
| --- | --- | --- | --- |
|31:16|Reserved|-|Reserved|
|15|CTSEN|R/W|CTS Hardware Flow Control Enable|
|14|RTSEN|R/W|RTS Hardware Flow Control Enable|
|13:12|Reserved|-|Reserved|
|11|RTS|R/W|Request to Send|
|10|DTR|R/W|Data Terminal Ready|
|9|RXE|R/W|Receive Enable|
|8|TXE|R/W|Transmit Enable|
|7|LBE|R/W|Loopback Enable|
|6:3|Reserved|-|Reserved|
|2:1|SIRLP|R/W|SIR Low Power Mode|
|0|UARTEN|R/W|UART Enable|

Sources: [src/pl011.rs(L12 - L31)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L12-L31)

## Hardware Features

### FIFO Configuration

The PL011 contains separate 16-entry FIFOs for transmit and receive operations, providing buffering to reduce interrupt overhead and improve system performance.

```mermaid
flowchart TD
subgraph Trigger_Levels["Trigger_Levels"]
    TX_1_8["TX: 1/8 (2 entries)"]
    TX_1_4["TX: 1/4 (4 entries)"]
    TX_1_2["TX: 1/2 (8 entries)"]
    TX_3_4["TX: 3/4 (12 entries)"]
    TX_7_8["TX: 7/8 (14 entries)"]
    RX_1_8["RX: 1/8 (2 entries)"]
    RX_1_4["RX: 1/4 (4 entries)"]
    RX_1_2["RX: 1/2 (8 entries)"]
    RX_3_4["RX: 3/4 (12 entries)"]
    RX_7_8["RX: 7/8 (14 entries)"]
end
subgraph FIFO_Architecture["FIFO_Architecture"]
    TXFIFO["TX FIFO16 entries8-bit width"]
    RXFIFO["RX FIFO16 entries8-bit width + 4-bit error"]
    TXLOGIC["TX Logic"]
    RXLOGIC["RX Logic"]
    IFLS_REG["IFLS RegisterTrigger Levels"]
end

IFLS_REG --> RXFIFO
IFLS_REG --> RX_1_8
IFLS_REG --> TXFIFO
IFLS_REG --> TX_1_8
RXLOGIC --> RXFIFO
TXFIFO --> TXLOGIC
```

Sources: [src/pl011.rs(L68 - L69)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L68-L69)

### Interrupt System

The PL011 provides comprehensive interrupt support with multiple interrupt sources and configurable masking.

|Interrupt|Bit|Description|
| --- | --- | --- |
|OEIM|10|Overrun Error Interrupt|
|BEIM|9|Break Error Interrupt|
|PEIM|8|Parity Error Interrupt|
|FEIM|7|Framing Error Interrupt|
|RTIM|6|Receive Timeout Interrupt|
|TXIM|5|Transmit Interrupt|
|RXIM|4|Receive Interrupt|
|DSRMIM|3|nUARTDSR Modem Interrupt|
|DCDMIM|2|nUARTDCD Modem Interrupt|
|CTSMIM|1|nUARTCTS Modem Interrupt|
|RIMIM|0|nUARTRI Modem Interrupt|

Sources: [src/pl011.rs(L71 - L72)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L71-L72) [src/pl011.rs(L94 - L96)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L94-L96) [src/pl011.rs(L100 - L101)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L100-L101)

## Signal Interface and Pin Configuration

### Primary UART Signals

The PL011 requires the following primary signals for basic UART operation:

|Signal|Direction|Description|
| --- | --- | --- |
|TXD|Output|Transmit Data|
|RXD|Input|Receive Data|
|UARTCLK|Input|UART Reference Clock|
|PCLK|Input|APB Bus Clock|
|PRESETn|Input|APB Reset (active low)|

### Modem Control Signals (Optional)

For full modem control functionality, additional signals are available:

|Signal|Direction|Description|
| --- | --- | --- |
|nRTS|Output|Request to Send (active low)|
|nCTS|Input|Clear to Send (active low)|
|nDTR|Output|Data Terminal Ready (active low)|
|nDSR|Input|Data Set Ready (active low)|
|nDCD|Input|Data Carrier Detect (active low)|
|nRI|Input|Ring Indicator (active low)|

```mermaid
flowchart TD
subgraph External_Device["External_Device"]
    EXT["External UART Deviceor Terminal"]
end
subgraph PL011_Pin_Interface["PL011_Pin_Interface"]
    UART["PL011 Controller"]
    TXD_PIN["TXD"]
    RXD_PIN["RXD"]
    RTS_PIN["nRTS"]
    CTS_PIN["nCTS"]
    DTR_PIN["nDTR"]
    DSR_PIN["nDSR"]
    DCD_PIN["nDCD"]
    RI_PIN["nRI"]
    CLK_PIN["UARTCLK"]
    PCLK_PIN["PCLK"]
    RST_PIN["PRESETn"]
end

CLK_PIN --> UART
CTS_PIN --> UART
DCD_PIN --> UART
DSR_PIN --> UART
EXT --> CTS_PIN
EXT --> RXD_PIN
PCLK_PIN --> UART
RI_PIN --> UART
RST_PIN --> UART
RTS_PIN --> EXT
RXD_PIN --> UART
TXD_PIN --> EXT
UART --> DTR_PIN
UART --> RTS_PIN
UART --> TXD_PIN
```

Sources: README.md references ARM documentation

## Hardware Initialization Requirements

### Power-On Reset Sequence

The PL011 requires a specific initialization sequence to ensure proper operation:

1. **Assert PRESETn**: Hold the reset signal low during power-up
2. **Clock Stabilization**: Ensure UARTCLK and PCLK are stable
3. **Release Reset**: Deassert PRESETn to begin initialization
4. **Register Configuration**: Configure control and interrupt registers

### Essential Configuration Steps

Based on the driver implementation, the hardware requires this initialization sequence:

```mermaid
flowchart TD
RESET["Hardware ResetPRESETn asserted"]
CLK_STABLE["Clock StabilizationUARTCLK, PCLK stable"]
REL_RESET["Release ResetPRESETn deasserted"]
CLEAR_INT["Clear All InterruptsICR = 0x7FF"]
SET_FIFO["Set FIFO LevelsIFLS = 0x00"]
EN_RX_INT["Enable RX InterruptIMSC = 0x10"]
EN_UART["Enable UART OperationCR = 0x301"]

CLEAR_INT --> SET_FIFO
CLK_STABLE --> REL_RESET
EN_RX_INT --> EN_UART
REL_RESET --> CLEAR_INT
RESET --> CLK_STABLE
SET_FIFO --> EN_RX_INT
```

The specific register values used in initialization:

* **ICR**: `0x7FF` - Clears all interrupt sources
* **IFLS**: `0x00` - Sets 1/8 trigger level for both TX and RX FIFOs
* **IMSC**: `0x10` - Enables receive interrupt (RXIM bit)
* **CR**: `0x301` - Enables UART, transmit, and receive (`UARTEN | TXE | RXE`)

Sources: [src/pl011.rs(L64 - L76)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L64-L76)

## Performance and Timing Considerations

### Clock Requirements

The PL011 operates with two clock domains:

* **PCLK**: APB bus clock for register access (typically system clock frequency)
* **UARTCLK**: Reference clock for baud rate generation (can be independent)

### Baud Rate Generation

The internal baud rate generator uses UARTCLK to create the desired communication speed. The relationship is:

```
Baud Rate = UARTCLK / (16 × (IBRD + FBRD/64))
```

Where IBRD and FBRD are integer and fractional baud rate divisors configured through additional registers not exposed in this implementation.

### FIFO Timing Characteristics

* **FIFO Depth**: 16 entries for both TX and RX
* **Access Time**: Single PCLK cycle for register access
* **Interrupt Latency**: Configurable based on FIFO trigger levels
* **Overrun Protection**: Hardware prevents data loss when FIFOs are properly managed

Sources: [src/pl011.rs(L68 - L69)&emsp;](https://github.com/arceos-org/arm_pl011/blob/a5a02f1f/src/pl011.rs#L68-L69) for FIFO configuration, ARM PL011 Technical Reference Manual for timing specifications