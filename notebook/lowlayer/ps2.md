---
layout: post
title: PS/2
---


## What is PS/2?
- PS/2 is a type of serial communication, typically used for user input devices (keyboard, mouse, bar code scanner, etc).
- It involves a controller (e.g. Intel 8042), the mechanical and electrical details of the communication itself, and a device.



## Intel 8042 (PS/2 Controller)
### Architecture Diagram
```
                                                    +----------------------+
                     +------------------------+     | first PS/2 connector |
    +---------+      |                        <----->      (keyboard)      |
    |         | Port |                        |     +----------------------+
    |   CPU   <------>        PS/2 KBC        |     +-----------------------+
    |         | 0x60 |                        <-----> second PS/2 connector |
    +-----^---+ 0x64 |                        |     |       (mouse)         |
          |          +--+----------------+----+     +-----------------------+
          |             |                |
          |             |                |
          |             |                |
          |     +-------v------+  +------v-------+
          |     |     IRQ1     |  |     IRQ12    |
          +-----+     PIC0 IRQ2<--+     PIC1     |
                |              |  |              |
                +--------------+  +--------------+
```


### I/O Ports

| I/O port | Access Type | Purpose          |
| -------- | ----------- | ---------------- |
| 0x60     | Read/Write  | Data             |
| 0x64     | Read        | Status register  |
| 0x64     | Write       | Command register |


### Status Register

| Bit | Description                                                                                                       |
| --- | ----------------------------------------------------------------------------------------------------------------- |
| 0   | Output buffer status (0 = empty, 1 = full). Must be set before attemping to read data from I/O port 0x60.         |
| 1   | Input buffer status (0 = empty, 1 = full). Must be clear before attemping to write data to I/O port 0x60 or 0x64. |
| 2   | System flag (meant to be cleared on reset and set by firmware if the system passes POST)                          |
| 3   | Command/data (0 = data written to input buffer is for PS/2 device, 1 = for PS/2 controller command)               |
| 4   | Unknown (chipset specific)                                                                                        |
| 5   | Send time-out error (0 = no error, 1 = time-out error)                                                            |
| 6   | Receive time-out error (0 = no error, 1 = time-out error)                                                         |
| 7   | Parity error (0 = no error, 1 = parity error)                                                                     |


### Commands
- The command port (I/O port 0x64) is used for sending commands to the PS/2 controller (not to PS/2 devices).
- If there is a "next byte" to be written, the next byte needs to be written to I/O port 0x60 after making sure that the controller is ready for it (by making sure bit 1 of the status register is clear).
- If there is a response byte, then the response byte needs to be read from I/O port 0x60 after making sure it has arrived (by making sure bit 0 of the status register is set).

| Command Byte | Description                                       | Response Byte                 |
| ------------ | ------------------------------------------------- | ----------------------------- |
| 0x20         | Read controller configuration byte                | Controller configuration byte |
| 0x60         | Write next byte to controller configuration byte  | None                          |
| 0xA7         | Disable second PS/2 port                          | None                          |
| 0xA8         | Enable second PS/2 port                           | None                          |
| 0xAD         | Disable first PS/2 port                           | None                          |
| 0xAE         | Enable first PS/2 port                            | None                          |
| 0xD0         | Read controller output port                       | None                          |
| 0xD1         | Write next byte to controller output port         | None                          |
| 0xD2         | Write next byte to first PS/2 port output buffer  | None                          |
| 0xD3         | Write next byte to second PS/2 port output buffer | None                          |
| 0xD4         | Write next byte to second PS/2 port input buffer  | None                          |


### Controller Configuration Byte

| Bit | Description                                             |
| --- | ------------------------------------------------------- |
| 0   | First PS/2 port interrupt (1 = enabled, 0 = disabled)   |
| 1   | Second PS/2 port interrupt (1 = enabled, 0 = disabled)  |
| 2   | System flag (1 = POST passed, 0 = POST failed)          |
| 3   | Should be zero                                          |
| 4   | Set 1 to disable PS/2 keyboard                          |
| 5   | Set 1 to disable PS/2 mouse                             |
| 6   | First PS/2 port translation (1 = enabled, 0 = disabled) |
| 7   | Must be zero                                            |


### Controller Output Port

| Bit | Description                                        |
| --- | -------------------------------------------------- |
| 0   | System reset                                       |
| 1   | A20 gate                                           |
| 2   | Second PS/2 port clock                             |
| 3   | Second PS/2 port data                              |
| 4   | Output buffer full with byte from first PS/2 port  |
| 5   | Output buffer full with byte from second PS/2 port |
| 6   | First PS/2 port clock                              |
| 7   | First PS/2 port data                               |



## PS/2 Keyboard
- The PS/2 keyboard is a device that talks to a PS/2 controller using serial communication.
- The PS/2 keyboard accepts commands and sends responses to those commands, and also sends scan codes indicating when a key was pressed or released.


### Commands
- A command is one byte.
- Some commands have data byte/s which must be sent after the command byte.
- The keyboard typically responds to a command by sending either an "ACK" (to acknowledge the command) or a "Resend" (to say something was wrong with the previous command) back.
- Don't forget to wait between the command, the data and the response from keyboard.

#### Special Bytes

| Response Byte | Description                |
| ------------- | -------------------------- |
| 0xFA          | Command acknowledged (ACK) |
| 0xFE          | Resend                     |


### Scan Code Sets
- A scan code set is a set of codes that determine when a key is pressed or repeated, or released.
- There are 3 different sets of scan codes.
	- The oldest is "scan code set 1".
	- The default is "scan code set 2".
	- There is a newer (more complex) "scan code set 3".
	- Modern keyboards should support all 3 scan code sets, however some don't. Scan code set 2 is the only scan code set that is guaranteed to be supported.



## PS/2 Mouse
### Commands

| Command Byte | Data Byte | Description            |
| ------------ | --------- | ---------------------- |
| 0xF5         | None      | Disable data reporting |
| 0xF4         | None      | Enable data reporting  |

| Response Byte | Description                |
| ------------- | -------------------------- |
| 0xFA          | Command acknowledged (ACK) |


### Data Format
- After enabling the PS/2 mouse, it will send the generic packet formatted as follows:
	- First byte
		- bit 0 - bl: Button left
		- bit 1 - br: Button right
		- bit 2 - bm: Button middle
		- bit 3 - ao: Always one
		- bit 4 - xs: X-axis sign bit
		- bit 5 - ys: Y-axis sign bit
		- bit 6 - xo: X-axis overflow
		- bit 7 - yo: Y-axis overflow
	- Second byte
		- xm: X-axis movement value
	- Third byte
		- ym: Y-axis movement value

- The absolute position can be calculated by accumulating data (relative offset) from the mouse.
- The relative offset is 9-bit value (sign bit + 8 bit value).
```
state = first_byte;

d = second_byte;
if ((state & (1 << 4)) > 0) {
	rel_x = d - 0x100;
}
abs_x += rel_x;

d = third_byte;
if ((state & (1 << 5)) > 0) {
	rel_y = d - 0x100;
}
abs_y += rel_y;
```



## Links
- [PS/2 - OSDev Wiki](https://wiki.osdev.org/PS/2)
- ["8042" PS/2 Controller - OSDev Wiki](https://wiki.osdev.org/%228042%22_PS/2_Controller)
- [PS/2 Keyboard - OSDev Wiki](https://wiki.osdev.org/PS/2_Keyboard)
- [PS/2 Mouse - OSDev Wiki](https://wiki.osdev.org/PS/2_Mouse)