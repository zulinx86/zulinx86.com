---
layout: post
title: PIC (Programmable Interrupt Controller)
---


## What is PIC?
- Whenever an interrupt occurs, the CPU suspends the currently running program and switches to the interrupt service routine (ISR).
- There are two types of interrupt:
	- Vectored interrupt
		- Its ISR address is known to the CPU.
		- In case of vectored interrupts, the CPU holds the address of the memory location where ISR is stored.
	- Non-vectored interrupt
		- In case of non-vectored interrupts, the CPU has to reach the ISR but it does not hold the address of ISR.
		- In this case, the device causing the interrupt provides the ISR address to the CPU.
- PIC combines multiple (non-vectored) interrupt lines into a common line and provide those interrupts to the CPU based on priority through the common line.
- Basicallu the external devices initially interrupt the PIC and further the PIC interrupts the CPU.


### Links
- [What is 8259 Programmable Interrupt Controller (PIC)? Definition, Need, Features and Architecture of 8259 - Electronics Desk](https://electronicsdesk.com/8259-programmable-interrupt-controller.html)



## Intel 8259 Family
- Intel 8259
	- It was designed for the Intel 8085 and Intel 8086 processors.
	- It was introduced as part of Intel's MCS 85 family in 1976.
- Intel 8259A
	- It was the interrupt controller for the ISA bus in the original IBM PC and IBM PC/AT.
	- It was introduced in the original PC in 1981 and maintained by the PC/XT in 1983.
	- The second 8259A was added with the introduction of the PC/AT.


### PC/AT
- Primary
	- IRQ0: system timer
	- IRQ1: keyboard
	- IRQ2: to secondary's INT pin
	- IRQ3: serial port (COM2)
	- IRQ4: serial port (COM1)
	- IRQ5: printer port (LPT2)
	- IRQ6: FDD controller
	- IRQ7: printer port (LPT1)
- Secondary
	- IRQ8: RTC
	- IRQ9: empty
	- IRQ10: empty
	- IRQ11: empty
	- IRQ12: PS/2 mouse
	- IRQ13: coprocessor
	- IRQ14: primary IDE controller
	- IRQ15: secondary IDE controller
- I/O Ports
	- Primary PIC
		- Command: 0x20
			- Read: IRR/ISR
			- Write: ICW1/OCW2,OCW3
		- Data: 0x21
			- Read: IMR
			- Write: ICW2,ICW3,ICW4/OCW1
	- Secondary PIC
		- Command: 0xA0
			- Read: IRR/ISR
			- Write: ICW1/OCW2,OCW3
		- Data: 0xA1
			- Read: IMR
			- Write: ICW2,ICW3,ICW4/OCW1s


### Pin Diagram
```
     +-------+
  -CS|1    28|VCC
  -WR|2    27|A0
  -RD|3    26|-INTA
   D7|4    25|IR7
   D6|5    24|IR6
   D5|6    23|IR5
   D4|7    22|IR4
   D3|8    21|IR3
   D2|9    20|IR2
   D1|10   19|IR1
   D0|11   18|IR0
 CAS0|12   17|INT
 CAS1|13   16|-SP/-EN
  GND|14   15|CAS2
     +-------+
```
- VCC: power supply
- GND: ground
- CS: chip select
	- A low on this pin enabled RD and WR communication between the CPU and the 8259A.
- WR: write
	- A low on this pin when CS is low enables the 8259A to accept command words from the CPU.
- RD: read
	- A low on this pin when CS is low enables the 8259A to release status onto the data bus for the CPU.
- D0-D7: bidirectional data bus
	- Control, status and interrupt-vector information is transferred via this bus.
- CAS0-CAS2: cancade lines
- SP/EN: secondary program / enable buffer
	- In non-buffered mode, it is used to specify whether it works as primary or secondary. (high: primary, low: secondary)
	- In buffered mode, it is used as an output to enable data bus.
- INT: interrupt pin
	- This pin goes high whenever a valid interrupt request is asserted.
	- It is used to interrupt the CPU, thus it is connected to the CPU's interrupt pin (INTR).
- IR0-IR7: interrupt request lines
	- An interrupt request is executed by raising an IR input (low to high), and holding it high until it is acknolwedged (edge triggered mode) or just by a high level on an IR input (level triggered mode).
- INTA: interrupt acknowledge
	- This pin is used to enable the 8259A interrupt-vector data onto the data bus by a sequence of interrupt acknowledge pulses issued by the CPU.
- A0: A0 address line
	- This pin acts in conjunction with the CS, WR and RD pins.
	- It is used by the 8259A to decipher various command words the CPU writes and status the CPU wishes to read.
	- It is typically connected to the CPU A0 address line. (A1 for 8086, 8088).


### Block Diagram
```
                               Internal Bus
             +-----------------+    |     +-----------------+
    D0-D7 <=>| Data Bus Buffer |<==>|     |  Control Logic  |<- -INTA
             +-----------------+    |     |                 |-> INT
                                  +-|-----+-----------------+
                                  | |      |       |       ^
                                  | |---------------------------
             +-----------------+  | |    ^ |       |       |  ^
   -RD/-WR ->| Read/Write      |<-+ |    | v       v       |  |
        A0 ->| Control Logic   |  | |  +-----+   +----+   +-----+
             + ----------------+  | |  | ISR |<=>| PR |<==| IRR |<- IR0-IR7
       -CS ------^                | |  +-----+   +----+   +-----+
             +-----------------+  | |     ^        ^        ^
 CAS0-CAS3 ->| Cascade Buffer  |  | |     |        |        |
   -SP/-EN ->| Compator        |<-+ |   +---------------------+
             +-----------------+    |<=>|        IMR          |
                                    |   +---------------------+
```
- Interrupt Request Register (IRR)
	- It stores the interrupt requests generated by the peripheral devices.
	- Those requested interrupts are waiting acknowledgement from the CPU.
- In-Service Register (ISR)
	- It indicates the interrupts which are currently being executed by the CPU and are waiting EOI from the CPU.
- Interrupt Mask Register (IMR)
	- It stores a mask of interrupts that should not be sent an acknowledgement.
- Data Bus Buffer
	- It is bidirectional 8 bit data bus buffer that interfaces with the internal bus of the CPU.
- Read/Write Control Logic
	- It works only when the CS pin is set to low.
	- It is responsible for the flow of data depending upon the inputs of RD and WR.
	- It holds initialization command word (ICW) register and operation command word (OCW) register inside.
- Control Logic
	- It is the center/heart of the Intel 8259 and controls the overall operation of the system by sending the INTR signal to the CPU.
	- It receives INTA signal by the CPU when the CPU demands for the address of the interrupt service routine. It is responsible for sending the address of the desired interrupt service routine through the data bus.
- Priority Resolver
	- It decides which interrupt should be executed first among all interrupt requests present in IRR based on their priority.
	- An interrupt with highest priority is set in ISR. Also it resets the interrupt level which has already been serviced in IRR.
- Cascade Buffer / Compartor
	- It stores and compares the IDs of all 8259A's used in the system.
	- In the primary mode, it acts as a cascaded buffer.
	- In the secondary mode, it acts as a comparator.


### Interrupt Sequence in 8086
1. One or more of the interrupt request lines (IR0-7) are raised high, setting the corresponding IRR bits.
1. The 8259A evaluates these requests, and sends an INT to the CPU, if appropriate.
1. The CPU acknowledges the INT and responds with an INTA pulse.
1. Upon receiving an INTA from the CPU, the highest priority ISR bit is set and the corresponding IRR bit is reset. The 8259A does not drive the data bus during this cycle.
1. The 8086 will initiate a second INTA pulse. During this pulse, the 8259A releases an 8-bit pointer onto the data bus where it is read by the CPU.
1. This completes the interrupt cycle. In the AEOI mode, the ISR bit is reset at the end of the second INTA. Otherwise, the ISR bit remains set until an appropriate EOI command is issued at the end of the interrupt subroutine.


### Programming the 8259A
- The 8259A accepts two types of command words generated by the CPU:
	- Initialization Command Words (ICWs)
	- Operation Command Words (OCWs)


### ICWs (Initialization Command Words)
```
                 +------+
                 | ICW1 |
                 +--+---+
                    v
                 +------+
                 | ICW2 |
                 +--+---+
                    v
           +------------------+
         +-+ IN CASCADE MODE? |
         | +--------+---------+
    NO   |          v YES (SNGL=0)
 (SNGL=1)|       +------+
         |       | ICW3 |
         |       +--+---+
         +--------->|
                    v
            +-----------------+
         +--+ IS ICW4 NEEDED? |
         |  +-------+---------+
     NO  |          v YES (IC4=1)
  (IC4=0)|       +------+
         |       | ICW4 |
         |       +--+---+
         +--------->|
                    v
          +--------------------+
          |   READY TO ACCEPT  |
          | INTERRUPT REQUESTS |
          +--------------------+
```

#### ICW1
```
  B7    B6    B5    B4    B3    B2    B1    B0
+-----+-----+-----+-----+-----+-----+-----+-----+
|  0  |  0  |  0  |  1  |LTIM | ADI |SNGL | IC4 |
+-----+-----+-----+-----+-----+-----+-----+-----+
```
- `LTIM` (Level Triggered Interrupt Mode)
	- 0: Edge triggered interrupt mode
	- 1: Level triggered interrupt mode
- `ADI` (Address Interval)
	- 0: Interval = 8
	- 1: Interval = 4
- `SNGL` (Single)
	- 0: Cascading mode
	- 1: Single mode (no ICW3 will be issued)
- `IC4`
	- 0: ICW4 is not needed
	- 1: ICW4 is needed

#### ICW2
```
  B7    B6    B5    B4    B3    B2    B1    B0
+-----+-----+-----+-----+-----+-----+-----+-----+
| T7  | T6  | T5  | T4  | T3  |  0  |  0  |  0  |
+-----+-----+-----+-----+-----+-----+-----+-----+
```
- `T[3:7]`: Interrupt vector address

#### ICW3
```
Primary PIC
  B7    B6    B5    B4    B3    B2    B1    B0
+-----+-----+-----+-----+-----+-----+-----+-----+
| S7  | S6  | S5  | S4  | S3  | S2  | S1  | S0  |
+-----+-----+-----+-----+-----+-----+-----+-----+

Secondary PIC
  B7    B6    B5    B4    B3    B2    B1    B0
+-----+-----+-----+-----+-----+-----+-----+-----+
|  0  |  0  |  0  |  0  |  0  | ID2 | ID1 | ID0 |
+-----+-----+-----+-----+-----+-----+-----+-----+
```
- In cascading mode (`SNGL` is set to 0 in ICW1), it will load the 8-bit register.
	- Primary PIC
		- 0: IR pin does not have a secondary PIC.
		- 1: IR pin has a secondary PIC.
	- Secondary PIC
		- It indicates which IR pin of primary PIC is connected to this secondary PIC.

#### ICW4
```
  B7    B6    B5    B4    B3    B2    B1    B0
+-----+-----+-----+-----+-----+-----+-----+-----+
|  0  |  0  |  0  |SFNM | BUF | M/S |AEOI | uPM |
+-----+-----+-----+-----+-----+-----+-----+-----+
```
- SFNM (Special Fully Nested Mode)
	- 0: Not special fully nested mode
	- 1: Special fully nested mode
- BUF & M/S
	- 0 & X: Non-buffered mode
	- 1 & 0: Buffered mode / Secondary
	- 1 & 1: Buffered mode / Primary
- AEOI
	- 0: Normal EOI
	- 1: Auto EOI
- uPM
	- 0: MCS-80/85 mode
	- 1: 8086/8088 mode


### OCWs (Operation Command Words)
#### OCW1
```
  B7    B6    B5    B4    B3    B2    B1    B0
+-----+-----+-----+-----+-----+-----+-----+-----+
| M7  | M6  | M5  | M4  | M3  | M2  | M1  | M0  |
+-----+-----+-----+-----+-----+-----+-----+-----+
```
- OCW1 sets and clears the mask bits in the IMR.
- M0-M7
	- 0: Indicates the interrupt is enabled.
	- 1: Indicates the interrupt is masked.

#### OCW2
```
  B7    B6    B5    B4    B3    B2    B1    B0
+-----+-----+-----+-----+-----+-----+-----+-----+
|  0  |  R  | SL  | EOI |  0  | L2  | L1  | L0  |
+-----+-----+-----+-----+-----+-----+-----+-----+
```
- R & SL & EOI
	- 0 & 0 & 1: Non-specific EOI
	- 0 & 1 & 1: Specific EOI
	- 1 & 0 & 1: Rotate on non-specific EOI
	- 1 & 0 & 0: Rotate in Auto EOI mode (set)
	- 0 & 0 & 0: Rotate in Auto EOI mode (clear)
	- 1 & 1 & 1: Rotate on specific EOI
	- 1 & 1 & 0: Set priority
	- 0 & 1 & 0: No operation
- L0-L2: IR pin to be acted upon

#### OCW3
```
  B7    B6    B5    B4    B3    B2    B1    B0
+-----+-----+-----+-----+-----+-----+-----+-----+
|  0  |ESMM | SMM |  0  |  1  |  P  | RR  | RIS  |
+-----+-----+-----+-----+-----+-----+-----+-----+
```
- ESMM (Enable Speciak Mask Mode)
	- 0: SMM bit becomes a "don't care'.
	- 1: It enables the SMM bit to set or reset the special mask mode.
- SMM (Special Mask Mode)
	- 0: It will revert to normal mask mode.
	- 1: It will enter special mask mode.
- P (Poll)
	- 0: No poll
	- 1: Poll
- RR (Read Register) & RIS (Register)
	- 0 & X: No action
	- 1 & 0: Read IRR
	- 1 & 1: Read ISR


### EOI (End Of Interrupt)
- EOI operations support the following:
	- Specific EOI
		- It specifies the IRQ level that it is acknowledging in the ISR.
	- Non-specific EOI
		- It resets the IRQ level in the ISR.
	- Auto EOI
		- It resets the IRQ level in the ISR immediately after the interrupt is acknowledged.


### Trigger Mode
- Edge trigger mode
	- An interrupt request will be recognized by a low to high transition on an IR pin.
	- The IR pin can remain high without generating another interrupt.
- Level trigger mode
	- An interrupt request will be recognized by a high level on IR pin, and there is no need for an edge detection.
	- The interrupt request must be removed before the EOI command is issued or the CPU interrupts is enabled to prevent a second interrupt from occurring.


### Code Examples
Common Definition
```c
#define PIC0				(0x20)
#define PIC0_COMM			(PIC0)
#define PIC0_DATA			(PIC0 + 1)
#define PIC1				(0xA0)
#define PIC1_COMM			(PIC1)
#define PIC1_DATA			(PIC1 + 1)
```

Initialization
```c
#define ICW1_ICW4			(0b00000001)
#define ICW1_SNGL			(0b00000010)
#define ICW1_INT4			(0b00000100)
#define ICW1_LTIM			(0b00001000)
#define ICW1_INIT			(0b00010000)

#define ICW2_MASTER_OFFSET	(0x20)
#define ICW2_SLAVE_OFFSET	(0x28)

#define ICW3_SLAVE_IRQ		2

#define ICW4_8086			(0b00000001)
#define ICW4_AUTO			(0b00000010)
#define ICW4_BUF_SLAVE		(0b00001000)
#define ICW4_BUF_MASTER		(0b00001100)
#define ICW4_SFNM			(0b00010000)

#define PIC0_IMR			(0b11111011)
#define PIC1_IMR			(0x11111111)

void pic_init()
{
	outb(PIC0_COMM, ICW1_INIT | ICW1_ICW4);
	outb(PIC0_DATA, ICW2_MASTER_OFFSET);
	outb(PIC0_DATA, 1 << ICW3_SLAVE_IRQ);
	outb(PIC0_DATA, ICW4_8086);

	outb(PIC1_COMM, ICW1_INIT | ICW1_ICW4);
	outb(PIC1_DATA, ICW2_SLAVE_OFFSET);
	outb(PIC1_DATA, ICW3_SLAVE_IRQ);
	outb(PIC1_DATA, ICW4_8086);

	outb(PIC0_DATA, PIC0_IMR);
	outb(PIC1_DATA, PIC1_IMR);
}
```

Non-Specific EOI
```c
#define OCW2_EOI	0x20

void pic_send_eoi(unsigned char irq)
{
	if (irq >= 8)
		outb(PIC1_COMM, OCW2_EOI);
	outb(PIC0_COMM, OCW2_EOI);
}
```

Specific EOI
```c
#define OCW2_EOI	0x60

void pic_send_eoi(unsigned char irq)
{
	if (irq >= 8) {
		outb(PIC1_COMM, OCW2_EOI | (irq - 0x08));
		outb(PIC0_COMM, OCW2_EOI | ICW3_SLAVE_IRQ);
	} else {
		outb(PIC0_COMM, OCW2_EOI | irq);
	}
}
```


### Links
- [8259 PIC Microprocessor - GeeksforGeeks](https://www.geeksforgeeks.org/8259-pic-microprocessor/)
- [Intel 8259 - Wikipedia](https://en.wikipedia.org/wiki/Intel_8259)
- [8259 PIC - OSDev Wiki](https://wiki.osdev.org/8259_PIC)
- [What is 8259 Programmable Interrupt Controller (PIC)? Definition, Need, Features and Architecture of 8259 - Electronics Desk](https://electronicsdesk.com/8259-programmable-interrupt-controller.html)
- [8259A.pdf](https://pdos.csail.mit.edu/6.828/2010/readings/hardware/8259A.pdf)
