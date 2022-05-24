---
layout: post
title: BIOS (Basic Input/Output System)
---


## INT 0x10 - Video BIOS Services
### AH = 0x00 - Set Video Mode
- Parameters
	- AL = video mode
    	- 0x12: 640x480 16 color graphics (VGA)
		- 0x13: 320x200 256 color graphics (VGA)


### AH = 0x0e - Write Text in Teletype Mode
- Parameters
	- AL = ASCII character to write
	- BH = page number (text modes)
	- BL = foreground pixel color (graphics modes)
- Return values: nothing


### AH = 0x11 - Character Generator Routine
#### AL = 0x30 - Get Current Character Generator Information
- Parameters
	- BH = information desired
		- 0x06: ROM 8x16 character table pointer
- Return values
	- CX = bytes per character
	- DL = rows
	- ES:BP = pointer to table



## INT 0x13 - Disk BIOS Services
### AH = 0x02 - Read Disk Sectors
- Parameters
	- AL = number of sectors to read
	- CH = cylinder number & 0xff
	- CL = (cylinder number & 0x300) \>\> 2 \| sector number (5 bits)
	- DH = head number
	- DL = drive number
	- ES:BX = pointer to buffer
- Return values
	- AH = status
	- AL = number of sectors read
	- CF = 0 (success), 1 (error)



## INT 0x16 - Keyboard BIOS Services
### AH = 0x02 - Read Keyboard Flags
- Parameters: nothing
- Return values
	- AL = BIOS keyboard flags
		- 0 bit: right shift key depressed
		- 1 bit: left shift key depressed
		- 2 bit: CTRL key depressed
		- 3 bit: ALT key depressed
		- 4 bit: scroll-lock is active
		- 5 bit: num-lock is active
		- 6 bit: caps-lock is active
		- 7 bit: insert is active



## Links
- [Interrupt Services DOS/BIOS/EMS/Mouse](https://stanislavs.org/helppc/idx_interrupt.html)