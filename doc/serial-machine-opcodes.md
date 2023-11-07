# Serial machine opcodes

The serial multiplex runs serial machine. Commands are taken from instructions
FIFO and executed. Synchronization with host is provided with an interrupt,
that can be raised from the code.

If any data are returned, those are placed to the output FIFO. It is
responsibility of instantiating code to read those data and process them.

> [!IMPORTANT]
> All multi-byte values are read from FIFOs and write to FIFOs in network order
> (big endian).

The following commands are supported:

## 0 - no op

No op, default. Don't do anything.

### Parameters

None.

### Retrurns

No data returned.

## 1 - write to the port

Write data to the port. Put them to output FIFO, data are then retrieved when
port is available for writing and transmitted on the port. If reply is expected
from the device, time needed to write the data shall be included in readout timeout.

### Parameters

| Length (bytes) | Description                                                            |
| -------------- | ---------------------------------------------------------------------- |
| 1              | Length of write buffer                                                 | 
| (parameter 1)  | Data to be written on the port                                         |

## 2 - read from the port with timeout in microseconds (us)

Read from the port. Reading stops either after timeout expires, or when given
number of character were already read.

### Parameters

| Length (bytes) | Description                                                            |
| -------------- | ---------------------------------------------------------------------- |
| 1              | Maximal number of bytes to read.                                       |
| 2              | Timeout value in microseconds.                                         |

### Returns

Readout data are returned as read from the port.

## 3 - read from the port with timeout in milliseconds (ms)

Read from the port. Reading stops either after timeout expires, or when given
number of character were already read. The command execution ends either with
timeout, or when expected number of bytes is read.

### Parameters

| Length (bytes) | Description                                                            |
| -------------- | ---------------------------------------------------------------------- |
| 1              | Maximal number of bytes to read.                                       |
| 2              | Timeout value in milliseconds.                                         |

### Returns

Readout data are returned as read from the port.

## 100 - wait microseconds (us)

Sleep for given numeber of microseconds. Code execution continues after wait is fulfilled.

### Parameters

| Length (bytes) | Description                                                            |
| -------------- | ---------------------------------------------------------------------- |
| 2              | Sleep time in microseconds.                                            |

## 101 - wait milliseconds (ms)

Sleep for given numeber of milliseconds. Code execution continues after wait is fulfilled.

### Parameters

| Length (bytes) | Description                                                            |
| -------------- | ---------------------------------------------------------------------- |
| 2              | Sleep time in milliseconds.                                            |

## 240 - raise interrupt

Raise serial port assigned interrupt.

> [!NOTE]
> This shall be used for host synchronization. Placing that after the last read
> command in sequence will enable host to read data after all relevant shall be processed.

### Parameters

None.

### Returns

No data returned.

## 253 - report serial port settings

Reports current serial port settings. Useful for debugging - it's not assumed
user will configure port settings, as those can be configured in LabVIEW before
creating bit-file.

### Parameters

None.

### Returns

| Length (bytes) | Description                                                            |
| -------------- | ---------------------------------------------------------------------- |
| 4              | Serial port speed in bauds.                                            |
| 2              | Serial port setttings. See below for details.                          |

### Serial port settings bitfield

| Bits  | Description                                                                     |
| ----- | ------------------------------------------------------------------------------- |
| 15-14 | Flow control. 0 - None, 1 - Software XON/XOFF, 2 - Hardware RTS/CTS.            |
| 13-12 | Stop bits. 1 - 1, 2 - 2, 3 - 1.5.                                               |
| 11-10 | Data bits. 0 - 5, 1 - 6, 2 - 7, 3 - 8.                                          |
| 9-7   | Parity. 0 - None, 1 - Odd, 2 - Even, 3 - Mark, 4 - Space.                       |
| 6-0   | Bytes waiting for read in port buffer.                                          |

## 254 - report telemetry

Auxiliary command, comes handy for port statistics/debugging. Use opcode 255 to
reset counter values.

### Parameters

None.

### Returns

| Length (bytes) | Description                                                            |
| -------------- | ---------------------------------------------------------------------- |
| 4              | Counter of written bytes.                                              |
| 4              | Counter of read bytes.                                                 |
| 2              | read timeouts. Counts read command ended due to timeouts.              |

## 255 - clear counters, flush FIFOs

Clear all counters and port buffers.

### Parameters

None.

### Returns

No data returned.
