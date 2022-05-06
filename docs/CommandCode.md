# FIFO commands

Communication between FPGA and C++ CSC code occurs on Host->Target and
Target->Host FIFOs. Hosts (CSC) sends command on some FIFO, and FPGA module
responds on another predefined FIFO.

# CommandsFIFO codes

| Code | Parameter(s)            | Description                                                         |
| ---- | ----------------------- | ------------------------------------------------------------------- |
| 17   | 32 bit float            | Sets mixing valve. Single parameter is requested current.           |
| 21   | Length, Modbus commands | FCU Modbus commands.                                                |
| 62   | Off == 0                | Switch GIS heartbeat.                                               |
| 63   | Off == 0                | Power on/off VFD/Glycol pump.                                       |
| 252  |                         | Clear Modbus IRQ, start modbus communication.                       |

# RequestFIFO codes

| Code | Description                                                                    |
| ---- | ------------------------------------------------------------------------------ |
| 9    | Request mixing valve position. Mixing valve position is written to SGL output. |
| 25   | Sends to U16ResponseFIFO Modbus responses.                                     |
| 77   | Sends last valid Glycol loop line to U8ResponseFIFO.                           |
| 78   | Sends curret Glycol temperature line to U8ResponseFIFO.                        |
| 79   | Sends 8 Glycol temperature values to SGLResponseFIFO.                          |


# MPUCommandFIFO codes

Allows access to MPU (Modbus Processing Unit) based devices. Those are flow
meter and VFD control of the glycol pump. MPU programming are described 

| Code | Parameter(s)          | Description                                                         |
| ---- | --------------------- | ------------------------------------------------------------------- |
| 1    | Length, MPU programme | MPU code for VFD.                                                   |
| 2    | Length, MPU programme | MPU code for FlowMeter.                                             |

