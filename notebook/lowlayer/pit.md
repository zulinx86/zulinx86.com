---
layout: post
title: Programmable Interval Timer (PIT)
---

## What is PIT?
- A PIT is a counter/timer device that generates an output signal when it reaches a programmed count.
- The Intel 8253 and 8254 are programmable interval timers (PITs).



## Intel 8253 / 8254
- It basically consists of an oscillator, a prescaler and 3 independent frequency dividers.
- Each frequency divider has an output, which is used to allow the timer to control external circuit.
- Oscillator
	- The oscillator runs at (roughly) 1.193182 MHz.
- Frequency dividers
	- The basic principle of a frequency divider is to divide a frequency to obtain a slower frequency. This is typically done by using a counter. Each pulse from the input frequency causes the counter to be decreased, and when that counter has reached zero a pulse is generated on the output and the counter is reset.
	- The reset counter has only 16 bits (0 - 65535).
	- The PIT chip has three separate frequency dividers (channels) that are programmable.


### I/O Ports

| I/O port | Description                                        |
| -------- | -------------------------------------------------- |
| 0x40     | Channel 0 data port (read/write)                   |
| 0x41     | Channel 1 data port (read/write)                   |
| 0x42     | Channel 2 data port (read/write)                   |
| 0x43     | Mode/Command register (write only)                 |


### Pin Diagram
```
     +-------+
   D7|1    24|VCC
   D6|2    23|-WR
   D5|3    22|-RD
   D4|4    21|-CS
   D3|5    20|A1
   D2|6    19|A0
   D1|7    18|CLK2
   D0|8    17|OUT2
 CLK0|9    16|GATE2
 OUT0|10   15|CLK1
GATE0|11   14|GATE1
  GND|12   13|OUT1
     +-------+
```
- D0-D7: Data
	- Bi-directional three state data bus lines, connected to system data bus.
- CLK0: Clock 0
	- Clock input of counter 0.
- OUT0: Output 0
	- Output of counter 0.
- GATE0: Gate 0
	- Gate input of counter 0.
- GND: Ground
	- Power supply connection.
- OUT1: Output 1
	- Output of counter 1.
- GATE1: Gate 1
	- Gate input of counter 1.
- CLK1: Clock 1
	- Clock input of counter 1.
- GATE2: Gate 2
	- Gate input of counter 2.
- OUT2: Output 2
	- Output of counter 2.
- CLK2: Clock 2
	- Clock input of counter 2.
- A0, A1: Address
	- Used to select of the three counters or the control word register for read/write operations.
	- 0 & 0: counter 0
	- 0 & 1: counter 1
	- 1 & 0: counter 2
	- 1 & 1: control word register
- -CS: Chip select
	- A low on this input enables the PIT to responds to -RD and -WR signals.
- -RD: Read control
	- This input is low during CPU read operations.
- -WR: Write control
	- This input is low during CPU write operations.
- VCC: Power
	- +5V power supply connection.


### Block Diagram
```
                      Internal Bus
                             |
                             |
         +--------------+    |    +-----------+
 D0-7 <=>|   Data Bus   |<==>|    |           |<- CLK0
         |    Buffer    |    |<==>| Counter 0 |<- GATE0
         +------+-------+    |    |           |-> OUT0
                |            |    +-----------+
         +------+-------+    |          ^
   -RD ->|              |    |  +-------+
   -WR ->|              |    |  | +-----------+
    A0 ->|  Read/Write  |    |  | |           |<- CLK1
    A1 ->|    Logic     |    |<=|>| Counter 1 |<- GATE1
         |              |    |  | |           |-> OUT1
   -CS ->|              +-+  |  | +-----------+
         +--------------+ |  |  |       ^
                          |  |  +-------+
         +--------------+ |  |  | +-----------+
         | Control Word |<+  |  | |           |<- CLK2
         |   Register   |<===|<=|>| Counter 2 |<- GATE2
         +------+-------+    |  | |           |-> OUT2
                |            |  | +-----------+
                |            |  |       ^
                +------------|--+-------+
                             |
```
- Data Bus Buffer
	- This 3-state, bi-directional, 8-bit bufer is used to interface the PIT to the system bus.
- Read/Write Logic
	- It accepts inputs from the system bus and generates control signals for the other functional blocks of the PIT.
	- A0 and A1 select one of the three counters or the control word register.
	- A low on the -RD input tells the PIT that the CPU is reading one of the counters.
	- A low on the -WR input tells the PIT that the CPU iws wrigin either a control word or an initial count.
- Control Word Register
	- It is selected by the read/write logic when A0 = 1 and A1 = 1.
	- If the CPU does a write operation to the PIT, the data is stored in the control word register and is interpreted as a control word used to define the operation of the counters.
- Counter 0, 1, 2
	- These three functional blocks are identical in operation.
	- The counters are fully independent. Each counter may operate in a different mode.


### Mode/Command Register
<table>
	<tr>
		<th>Bits</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>7 & 6</td>
		<td>
			Select channel:<br>
			0 0 = channel 0<br>
			0 1 = channel 1<br>
			1 0 = channel 2<br>
			1 1 = read-back command
		</td>
	</tr>
	<tr>
		<td>5 & 4</td>
		<td>
			Access mode:<br>
			0 0 = latch count value command<br>
			0 1 = access mode: lobyte only<br>
			1 0 = access mode: hibyte only<br>
			1 1 = access mode: lobyte/hibyte
		</td>
	</tr>
	<tr>
		<td>3 & 2 & 1</td>
		<td>
			Operating mode:<br>
			0 0 0 = mode 0 (interrupt on terminal count)<br>
			0 0 1 = mode 1 (hardware re-triggerable one-shot)<br>
			x 1 0 = mode 2 (rate generator)<br>
			x 1 1 = mode 3 (square wave generator)<br>
			1 0 0 = mode 4 (software triggered strobe)<br>
			1 0 1 = mode 5 (hardware triggered strobe)
		</td>
	</tr>
	<tr>
		<td>0</td>
		<td>
			BCD/Binary mode:<br>
			0 = binary counter 16-bits<br>
			1 = BCD counter (4 decades)
		</td>
	</tr>
</table>


### Programming the PIT
- Counters are programmed by writing a control word and then an initial count.
	- The control words are written into the control word register. The control word itself specifies which counter is being programmed.
	- Initial counts are written into the counters, not the control word register. The format of the initial count is determined by the control word used.

#### Write Operations
- Only two conventions need to be remembered:
	1. For each counter, the control word must be written before the initial count is written.
	1. The initial count must follow the count format specified in the control word.

#### Read Operations
- There are three possible methods for reading the counters
	- Simple read operation
	- Counter latch command
	- Read-back command


### Mode Definitions
- CLK pulse: a rising edge, then a falling edge, in that order, of a counter's CLK input.
- Trigger: a rising edge of a counter's GATE input.
- Counter loading: the transfer of a count from the CR (Counter Register) to the CE (Counting Element).

#### Mode 2: Rate Generator
- This mode functions like a divide-by-N counter.
- It is typically used to generate a real time clock interrupt.
- OUT will initially be high. When the initial count has decremented to 1, OUT goes low for one CLK pulse. OUT then goes high again, the counter reloads the initial count and the process is repeated.
- GATE = 1 enables counting; GATE = 0 disables counting.


## Links
- [Programmable interval timer - Wikipedia](https://en.wikipedia.org/wiki/Programmable_interval_timer)
- [Intel 8253 - Wikipedia](https://en.wikipedia.org/wiki/Intel_8253)
- [Programmable Interval Timer - OSDev Wiki](https://wiki.osdev.org/Programmable_Interval_Timer)