# ts_M1M3ThermalFPGA

## Description

This repository contains the LabVIEW 2018 FPGA design for the M1M3 Thermal
System FPGA used by the **ts_M1M3Thermal** software. The FPGA is middleman
between [C/C++ control application](https://github.com/lsst-ts/ts_m1m3thermal)
and hardware connected to cRIO. 

:warning: Please use **git clone --recursive** to get linked dependencies (Common_ libraries).

## LabView Dependencies

* FPGA Support
* cRIO support
* Real Time support
* [NI Tools Network](https://www.ni.com/labview-tools-network)
  * LabView FPGA Floating-Point Library by NI

If all is installed and setup properly, LabView splash screen will show chip
and clock icons - see [an
example](https://www.evergreeninnovations.co/blog-labview-rt-project/).

:warning: Please use **git clone --recursive** to get linked dependencies (Common_ libraries).

## Build Instructions

Building the FPGA takes just about 45 minutes. As [C++
Controller](https://github.com/lsst-ts/ts_m1m3support) is used to talk to FPGA,
you need to generate C API and transfer the bitfile to cRIO, and C header and
source files to src/LSST/M1M3/SS/FPGA directory. Bitfile is loaded by
NiFpga_Open call, and contains binary data send to program the FPGA.

It is common for the FPGA build process to get stuck on the *Generate Xilinx
IP* step. To restart the build, kill all Xilinx processes from task manager.

1. Open LabVIEW 2018.
2. Open ts_M1M3ThermalFPGA.lvproj
3. Expand RT CompactRIO Target
4. Expand FPGA Target
5. Expand Build Specifications
6. Select ts_M1M3ThermalFPGA
7. Right-Click -> Build
8. Select "Use the local compile server" _(it's usually faster than LabView FPGA Compile Cloud)_
9. Click OK
10. Wait for build to successfully finish
11. Select ts_M1M3ThermalFPGA.vi (under FPGA Target)
12. Right click, select **"Launch C API Generator"**
13. Click **Generate** (after selecting existing output directory and leaving Prefix blank)
14. Copy resulting lvbitx file to ts_m1m3thermal/Bitfiles, and NiFpga_M1M3ThermalFPGA.h to ts_m1m3Thermal/src/NiFpga
15. Recompile ts_M1M3Thermal

## Overview

The FPGA design makes heavy use of FIFOs and **S**ingle **C**ycle **T**imed
**L**oop (**SCTL**) for critical hardware loops. As FPGA has to sample DIOs
(serial lines, pumps, flow meters,.. ), it's critical those are properly timed
and running all the time. In a classic front/back processing design (see e.g.
Linux Kernel interrupt handling), sampling code just record the values or bits
and bytes and ship them to a FIFO queues for handling. SCTLs, which are
guarantee to take one clock tick, are ideal for handling inputs and outputs.

Receiving (reading) code works by placing values into FIFO. Transmitting
(writing) code works by reading (without timeout) from FIFO, and if something
is available, act accordingly. This is coupled with DIO states (e.g. to make
sure next bit/byte on serial line is transmitted after the current).

## Command multiplexing

Commands, followed by arguments are filled into _Software
Resources/CommandFIFO_. _Commands/CommandFIFOMultiplexer_ read this and
multiplex the command to various handlers _(This is exactly how CPUs

handle instructions from binary code)_. Handlers fills in queues (e.g. ModBus

writes queue with instructions to write if you are writing to a queue).
**SCTL**s are handling low level IO. 

## Request multiplexing

Data cannot be read by controller directly (via DMA), but has to be passed
through FIFOs _(usually FPGAs allows DMA to read data directly from memory, but
that doesn't seem to be cause with cRIO design)_. So Controller have to issue
request by writing it to _Software Resources/RequestFIFO_. Responses are read
from _Software Resources/SGLResponseFIFO_, _Software Resources/U8ResponseFIFO_
and _Software Resources/U16ResponseFIFO_.

## Telemetry, Health and Status

Telemetry or Health and Status data are recorded in **STCL**. Telemetry request
(253, _Data Types/Addresses_) dumps 323 U8 values from
_Telemetry/Hardware/Memory_ into _Software Resources/U8ResponseFIFO_. _Memory_
is filled from various FIFOs, which are filled from DIOs - see _Telemetry_.

## Health and Status

See [HealthAndStatusMemory.md](HealthAndStatusMemory.md) for memory content.
To request memory data, write command followed by parameter into
HealthAndStatusControlFIFO. Response to command is written into
HealthAndStatusDataFIFO (U64).

### Health and Status commands

| Command | Parameter | Action                                                            |
| ------- | --------- | ----------------------------------------------------------------  |
|  1      | Address   | Write address content (single U64) into HealthAndStatusDataFIFO   |
|  2      |  N/A      | Write 64 U64 into HealthAndStatusDataFIFO. This is memory content |
|  3      |  N/A      | Clear memory - write 0 to all memory cells                        |

### Health and Status Memory

See [HealthAndStatusMemory.md](HealthAndStatusMemory.md) for memory content.

## Digital Input

**DigitalInput** is very simple process that takes the digital input signals
and samples them every 0.200 ms (5kHz). The trigger for the process can be
found under *DigitalInput/Support/DigitalInputTrigger.vi* which produces a
trigger every 0.200ms and then waits for the sample process to complete. Once a
trigger is produced the *DigitalInput/Support/DigitalInputSampleLoop.vi* will
read the current timestamp and state of all digital inputs and place that
sample into a FIFO. Then at some other point in time the
*Telemetry/Support/TelemetryUpdate.vi* will call the
*DigitalInput/TryUpdateDigitalInputSample.vi* to read from **DigitalInputFIFO**
Sample, Timestamp and Value fields and writes those into three entries in
**DigitalOutputTelemetryFIFO**. **DigitalOutputTelemetryFIFO** is copied into
**TelemetryFIFO**. **TelemetryFIFO** is processed in
*Telemetry/Support/TelemetryMemoryUpdate.vi*, with values written into
*Telemetry/Hardware/Memory*. Once processed, registers *TelemetryEmptyRegister*
and *DigitalTelemetryEmpty* register are true and new sample can be obtained.

Since a SCTL only allows a FIFO writes in one loop and reads in another loop
the design utilizes multiple FIFOs to get around this restriction. In the
example above a digital input sample is pushed into the
*DigitalInputTelemetryFIFO* so that the it can be read by the
*Telemetry/CopyToTelemetryFIFO.vi* loop and pushed into the global
*TelemetryFIFO* which doesn't run inside a SCTL.

The fan coil units (FCUs)  modbus processes are much more complex and rely on
the host machine to parse the data.

# DIO assignment

## Slot 1 - [NI 9207](https://www.ni.com/en-us/support/model.ni-9207.html)

| Port | Assignment |
| ---- | ---------- |
| AI0  | CT7 current |
| AI1  | CT8 current |
| AI2  |            |
| AI3  |            |
| AI4  |            |
| AI5  |            |
| AI6  |            |
| AI7  |            |
| AI8  | Mixing Valve position |
| AI9  |            |
| AI10 |            |
| AI11 |            |
| AI12 |            |
| AI13 |            |
| AI14 |            |
| AI15 |            |

## Slot 2 - [NI 9265](https://www.ni.com/en-us/support/model.ni-9265.html)

| Port | Assignment |
| ---- | ---------- |
| AO0  | Mixing valve command |
| AO1  |            |
| AO2  |            |
| AO3  |            |

## Slot 3 - [NI 9401](https://www.ni.com/en-us/support/model.ni-9401.html)

:bus: ModBus A

| Port | Assignment |
| ---- | ---------- |
| DIO0 | bus A Rx   |
| DIO1 | -          |
| DIO2 | -          |
| DIO3 | -          |
| DIO4 | bus A Tx   |
| DIO5 | -          |
| DIO6 | -          |
| DIO7 | -          |

## Slot 4 - [NI 9425](https://www.ni.com/en-us/support/model.ni-9425.html)

| Port | Assignment |
| ---- | ---------- |
| DI0  | PS14 Status |
| DI1  | PS15 Status |
| DI2  | PS16 Status |
| DI3  | Control Redunancy Status |
| DI4  | Fan Coil Diffuser Status |
| DI5  | AC Power CB15 Status |
| DI6  | Utility Outlet CB 18 Status |
| DI7  | Coolant Pump OL Status |
| DI8  |            |
| DI9  |            |
| DI10 |            |
| DI11 |            |
| DI12 |            |
| DI13 |            |
| DI14 |            |
| DI15 |            |
| DI16 | FC Heaters Off Interlock |
| DI17 | Coolant Pump Off Interlock |
| DI18 | GIS HB Lost Interlock |
| DI19 | Mixing Valve Closed Interlock |
| DI20 | Support System HB Lost Interlock |
| DI21 | Cell Door Open Interlock |
| DI22 | GIS Eathquake Interlock |
| DI23 | Coolant pump E Stop Interlock |
| DI24 | Cabinet Over Temperature Interlock |
| DI25 |            |
| DI26 |            |
| DI27 |            |
| DI28 |            |
| DI29 |            |
| DI30 |            |
| DI31 |            |

## Slot 5 - [NI 9485](https://www.ni.com/en-us/support/model.ni-9485.html)

| Port | Assignment |
| ---- | ---------- |
| CH0  | Fan Coils Heaters On |
| CH1  | Thermal Heartbeat |
| CH2  | Coolant Pump On |
| CH3  |            |
| CH4  |            |
| CH5  |            |
| CH6  |            |
| CH7  |            |

## Slot 6 - [NI 9871](https://www.ni.com/en-us/support/model.ni-9871.html)

| Port | Assignment |
| ---- | ---------- |
| 1    | VFD        |
| 2    | Flow Meter |
| 3    | Wind Sensor|
| 4    |            |

## Slot 7 - [NI 9870](https://www.ni.com/en-us/support/model.ni-9870.html)

| Port | Assignment |
| ---- | ---------- |
| 1    |            |
| 2    |            |
| 3    |            |
| 4    |            |

## Slot 8 - [NI 9213](https://www.ni.com/en-us/support/model.ni-9213.html)

| Port | Assignment |
| ---- | ---------- |
| CH0  | Air Mirror Top Surface |
| CH1  | Mirror Cell Air |
| CH2  | Mirror Coolant Supply |
| CH3  | Mirror Coolant Return |
| CH4  | Telescope Coolant Supply |
| CH5  | Telescope Coolant Return |
| CH6  |            |
| CH7  |            |
