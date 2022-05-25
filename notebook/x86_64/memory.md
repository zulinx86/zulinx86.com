---
layout: post
title: Memory Management
---



## Overview
- The memory management facilities of the IA-32 architecture are divided into two parts:
	- Segmentation
		- It provides a mechanism of isolating individual code, data, and stack modules so that multiple programs can run on the same processor without interfering with one another.
	- Paging
		- It provides a mechanism for implementing a conventional demand-paged, virtual-memory system where sections of a program's execution environment are mapped into physical memory as needed.
		_ It can also be used to provide isolation between multiple tasks.
- The use of segmentation can not be disabled. The use of paging is optional.

```
+-----------------+ Segmentation +----------------+    Paging    +------------------+
| Logical Address |------------->| Linear Address |------------->| Physical Address |
+-----------------+              +----------------+              +------------------+



      Logical Address (or Far Pointer)             Linear Address Space                                                    Physical Address Space
+--------------------+ +--------------------+     +--------------------+                                                   +--------------------+
|  Segment Selector  | |       Offset       |     |                    |         Linear Address                            |                    |
+---------+----------+ +----------------+---+     |                    |    +-----+-------+--------+                       |                    |
          |                             |         |                    | +->| Dir | Table | Offset |                       |                    |
          |     Global Descriptor       |         |                    | |  +-+---+---+---+---+----+                       |                    |
          |        Table (GDT)          |         |      Segment       | |    |       |       |                            |                    |
          |  +--------------------+     |         +--------------------+ |    |       |       |                            |                    |
          |  |        ...         |     |         |                    | |    |       |       +-------------------------+  |        Page        |
          |  +--------------------+     |         |        Page        | |    |       |                   Page Table    |  |- - - - - - - - - - |
          +->| Segment Descriptor +--+  |         |- - - - - - - - - - | |    |       |                +--------------+ +->|  Physical Address  |
             +--------------------+  |  |         |                    | |    |       |                |              | |  |                    |
             |        ...         |  |  +----+--->|   Linear Address   +-+    |       +-------------+->|    Entry     +-+->|- - - - - - - - - - |
             +--------------------+  |       |    |- - - - - - - - - - |      |   Page Directory    |  |              |    |                    |
                                     |       |    |                    |      |  +--------------+ +-+->+--------------+    |                    |
                                     +-------+--->+--------------------+      |  |              | |                        |                    |
                                     Segment Base |                    |      +->|    Entry     +-+                        |                    |
                                       Address    |                    |      |  |              |                          |                    |
                                                  |                    |    +-+->+--------------+                          |                    |
                                                  +--------------------+    |                                              +--------------------+
                                                                         +--+--+
                                                                         | CR3 |
                                                                         +-----+
```



## Segmentation
- Segmentation provides a mechanism for dividing the linear address space into segments.
- Segments can be used to hold the code, data, and stack for a program or to hold system data structures (such as a TSS or LDT).
- The processor enforces the boundaries between these segments and ensures that one program does not interfere with the execution of another program by writing into the other program's segments.
- The segmentation mechanism allows typing of segments so that the operations that may be performed on particular type of segment can be restricted.


### Memory Model
- Basic flat model
	- At least two segment descriptor must be created, one for a code segment and one for a data segment.
	- Both of these segments are mapped to the entire linear address space: that is, both segment descriptors have the same base address value of 0 and the same segment limit of 4 GiB.
	- ROM is generally located at the top of the physical address space, because the processor begins execution at 0xFFFFFFF0.
- Protected flat model
	- It is similar to the basic flat model, except the segment limits are set to include only the range of addresses for which physical memory actually exists.
	- A general-protection exception (#GP) is then generated on any attempt to access nonexistent memory.
	- More complexity can be added to this protected flat model to provide more protection.
		- For the paging mechanism to provide isolation between user and supervisor code and data, four segments need to be defined: code and data segments at privilege level 3 for the user, and code and data segments at privilege level 0 for the supervisor.
		- This flat segmentation model along with a simple paging structure can protect the oprating system from applications, and by adding a separate paging structure for each task, it can also protect applications from each other.
- Multi-segment model
	- It uses the full capabilities of the segmentation mechanism to provide hardware enforced protection of code, data structures, and programs and tasks.
	- Each program (or task) is given its own table of segment descriptors and its own segments.


### Segmentation in IA-32e Mode
- In IA-32e mode, the effects of segmentation depend on whether the processor is running in compatibility mode or 64-bit mode.
	- In compatibility mode, segmentation functions just as it does using legacy 16-bit or 32-bit protected mode semantics.
	- In 64-bit mode, segmentation is generally (but not completely) disabled, creating a flat 64-bit linear-address space.
		- The processor treats the segment base of CS, DS, ES, SS as zero, creating a linear address that is equal to the effective address.
		- The FS and GS segments can be used as additional base registers in linear address calculations.
		- Note that the processor does not perform segment limit checks at runtime in 64-bit mode.


### Translation from Logical Address to Linear Addresses
- A logical address (also called a far pointer) consists of a 16-bit segment selector and a 32-bit offset.
- To translate a logical address into a linear address, the processor does the following:	
	- The segment selector is a unique identifier for a segment and provides an offset into a descriptor table (GDT or LDT) to a segment descriptor.
	- The segment descriptor contains the base address in the linear address space, the segment size, the access rights, the privilege level and the segment type.
	- The offset part of the logical address is added to the base address for the segment to locate a byte within the segment.
	- In short, the base address plus the offset forms a linear address.
- In IA-32e mode, an Intel 64 processor uses the steps described above.
- In 64-bit mode, the offset and the base address of the segment are 64 bits instead of 32 bits.


### Segment Selectors
```
 15   03 02 01 00
+-------+--+-----+
| Index |TI| RPL |
+-------+--+-----+
```

<table>
	<tr>
		<th>Bit</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>15:03</td>
		<td>
			Index<br>
			Select one of 8192 descriptors in the GDT or LDT.<br>
			The processor mutiplies the index value by 8 and adds the result to the base address of the GDT or LDT from GDTR and LDTR.
		</td>
	</tr>
	<tr>
		<td>02</td>
		<td>
			TI (Table Indicator) flag<br>
			Specifies the descriptor table to use. (0 = GDT, 1 = LDT)
		</td>
	</tr>
	<tr>
		<td>01:00</td>
		<td>
			Request Privileged Level (RPL)<br>
			Specifies the privilege level of the selector<br>
		</td>
	</tr>
</table>


### Segment Descriptors
```
 63               56  55  54  53  52  51                48  47  46  45  44  43   40  39               32
+--------------------+---+---+---+---+---------------------+---+-------+---+--------+--------------------+
| Base Address 31:24 | G |D/B| L |AVL| Segment Limit 19:16 | P |  DPL  | S |  Type  | Base Address 23:16 |
+--------------------+---+---+---+---+---------------------+---+-------+---+--------+--------------------+
 31                                                     16  15                                        00
+----------------------------------------------------------+---------------------------------------------+
|                    Base Address 15:00                    |              Segment Limit 15:00            |
+----------------------------------------------------------+---------------------------------------------+
```

<table>
	<tr>
		<th>Field</th>
		<th>Size</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>Segment Limit</td>
		<td>20 bits</td>
		<td>
			Specifies the size of the segment.<br>
			The interpretation depends on the G flag:<br>
			- If the G flag is clear, the segment size can range from 1 B to 1 MiB, in byte increments.<br>
			- If the G flag is set, the segment size can range from 4 KiB to 4 GiB, in 4 KiB increments.<br>
		</td>
	</tr>
	<tr>
		<td>Base Address</td>
		<td>32 bits</td>
		<td>
			Defines the location of byte 0 of the segment within the 4 KiB linear address space.
		</td>
	</tr>
	<tr>
		<td>Type</td>
		<td>4 bits</td>
		<td>
			Indicates the segment or gate type and specifies the kinds of access and the direction of growth.
		</td>
	</tr>
	<tr>
		<td>S (Descriptor Type)</td>
		<td>1 bit</td>
		<td>
			Specifies whether the segment descriptor is for a system segment (S flag is clear) or a code or data segment (S flag is set).<br>
			The interpretation depends on the S flag.
		</td>
	</tr>
	<tr>
		<td>DPL (Descriptor Privilege Level)</td>
		<td>3 bits</td>
		<td>
			Specifies the privilege level of the segment.<br>
		</td>
	</tr>
	<tr>
		<td>P (Present)</td>
		<td>1 bit</td>
		<td>
			Indicates whether the segment is present in memory (set) or not present (clear).
		</td>
	</tr>
	<tr>
		<td>D/B</td>
		<td>1 bit</td>
		<td>
			Performs different functions depending on whether the segment descriptor is an executable code segment, an expand-down data segment, or a stack segment.<br>
			This flag should always be set to 1 for 32-bit code and data segments and to 0 for 16-bit code and data segments.
		</td>
	</tr>
	<tr>
		<td>G (Granularity)</td>
		<td>1 bit</td>
		<td>
			Determines the scaling of the segment limit.<br>
			- When this flag is clear, the segment limit is interpreted in byte units<br>
			- When this flag is set, the segment limit is interpreted in 4 KiB units.
		</td>
	</tr>
	<tr>
		<td>L (64-bit Code Segment)</td>
		<td>1 bit</td>
		<td>
			Indicates whether a code segment contains native 64-bit code in IA-32e mode.<br>
			- When this flag is clear, the instructions in this code segment are executed in compatibility mode.<br>
			- When this flag is set, the instructions in this code segment are executed in 64-bit mode.<br>
			If L is set, then D must be cleared. When not in IA-32e mode or for non-code segments, this flag is reserved and should always be set to 0.
		</td>
	</tr>
	<tr>
		<td>AVL</td>
		<td>1 bit</td>
		<td>This bit is available for use by system software.</td>
	</tr>
</table>

#### Code- and Data-Segment Descriptor Types
- When the S flag is set, the descriptor is for either a code or data segment.
- Code segments can be either conforming or non-conforming.
	- A transfer of execution into a more-privileged conforming segment allows execution to continue at the current privilege level.
	- A transfer of execution into a non-conforming segment at a different privilege level results in a general-protection exception (#GP), unless a call gate or task gate is used.
	- Execution cannot be transferred by a call or a jump to a less-privileged code segment, regardless of whether the target segment is a conforming or non-conforming segment. Attempting such an execution transfer will result in a general-protection exception.
- All data segments are non-conforming, meaning that they cannot accessed by less privileged programs or procedures.
	- Unlike code segments, data segments can be accessed by more privileged programs or procedures without using a special access gate.

<table>
	<tr><th>Bits</th><th>Descriptor Type</th><th>Description</th></tr>
	<tr><td>0b0000</td><td>Data</td><td>Read-Only</td></tr>
	<tr><td>0b0001</td><td>Data</td><td>Read-Only, Accessed</td></tr>
	<tr><td>0b0010</td><td>Data</td><td>Read/Write</td></tr>
	<tr><td>0b0011</td><td>Data</td><td>Read/Write, Accessed</td></tr>
	<tr><td>0b0100</td><td>Data</td><td>Read-Only, Expand-Down</td></tr>
	<tr><td>0b0101</td><td>Data</td><td>Read-Only, Expand-Down, Accessed</td></tr>
	<tr><td>0b0110</td><td>Data</td><td>Read/Write, Expand-Down</td></tr>
	<tr><td>0b0111</td><td>Data</td><td>Read/Write, Expand-Down, Accessed</td></tr>
	<tr><td>0b1000</td><td>Code</td><td>Execute-Only</td></tr>
	<tr><td>0b1001</td><td>Code</td><td>Execute-Only, Accessed</td></tr>
	<tr><td>0b1010</td><td>Code</td><td>Execute/Read</td></tr>
	<tr><td>0b1011</td><td>Code</td><td>Execute/Read, Accessed</td></tr>
	<tr><td>0b1100</td><td>Code</td><td>Execute-Only, Conforming</td></tr>
	<tr><td>0b1101</td><td>Code</td><td>Execute-Only, Conforming, Accessed</td></tr>
	<tr><td>0b1110</td><td>Code</td><td>Execute/Read, Conforming</td></tr>
	<tr><td>0b1111</td><td>Code</td><td>Execute/Read, Conforming, Accessed</td></tr>
</table>

#### System Descriptor Types
- When the S flag is clear, the descriptor type is a system descriptor.
- Types of system descriptors
	- System-segment descriptors
		- Local descriptor-table (LDT) segment descriptor
		- Task-state segment (TSS) descriptor
	- Gate descriptors
		- Call-gate descriptor
		- Interrupt-gate descriptor
		- Trap-gate descriptor
		- Task-gate descriptor

<table>
	<tr><th>Bits</th><th>32-Bit Mode</th><th>IA-32e Mode</th></tr>
	<tr><td>0b0000</td><td>Reserved</td><td>Reserved</td></tr>
	<tr><td>0b0001</td><td>16-bit TSS (Available)</td><td>Reserved</td></tr>
	<tr><td>0b0010</td><td>LDT</td><td>LDT</td></tr>
	<tr><td>0b0011</td><td>16-bit TSS (Busy)</td><td>Reserved</td></tr>
	<tr><td>0b0100</td><td>16-bit Call Gate</td><td>Reserved</td></tr>
	<tr><td>0b0101</td><td>Task Gate</td><td>Reserved</td></tr>
	<tr><td>0b0110</td><td>16-bit Interrupt Gate</td><td>Reserved</td></tr>
	<tr><td>0b0111</td><td>16-bit Trap Gate</td><td>Reserved</td></tr>
	<tr><td>0b1000</td><td>Reserved</td><td>Reserved</td></tr>
	<tr><td>0b1001</td><td>32-bit TSS (Available)</td><td>64-bit TSS (Available)</td></tr>
	<tr><td>0b1010</td><td>Reserved</td><td>Reserved</td></tr>
	<tr><td>0b1011</td><td>32-bit TSS (Busy)</td><td>64-bit TSS (Busy)</td></tr>
	<tr><td>0b1100</td><td>32-bit Call Gate</td><td>64-bit Call Gate</td></tr>
	<tr><td>0b1101</td><td>Reserved</td><td>Reserved</td></tr>
	<tr><td>0b1110</td><td>32-bit Interrupt Gate</td><td>64-bit Interrupt Gate</td></tr>
	<tr><td>0b1111</td><td>32-bit Trap Gate</td><td>64-bit Trap Gate</td></tr>
</table>

#### Call Gates
- Call gates facilitate controlled transfers of program control between different privilege levels.
- To access a call gate, a far pointer to the gate is provided as a target operand in a CALL or JMP instruction.
	- The segment selector from this far pointer identifies the call gate; the offset from this far pointer is required, but not used or checked by the processor (the offset can be set to any value).
	- linear address of the procedure entry point = base address from the code-segment descriptor + offset from the call gate.

```
 63                                                     48  47  46  45  44  43  40 39 37 36           32
+----------------------------------------------------------+---+-------+---+------+-----+----------------+
|                Entry Point in Segment 31:16              | P |  DPL  | 0 | 1100 | 000 |  Param. Count  |
+----------------------------------------------------------+---+-------+---+------+-----+----------------+
 31                                                     16  15                                        00
+----------------------------------------------------------+---------------------------------------------+
|                     Segment Selector                     |         Entry Point in Segment 15:00        |
+----------------------------------------------------------+---------------------------------------------+
```

<table>
	<tr>
		<th>Field</th>
		<th>Size</th>
		<th>Descriptor</th>
	</tr>
	<tr>
		<td>Entry Point</td>
		<td>32 bits</td>
		<td>Specifies an offset in the segment pointing to an entry point.</td>
	</tr>
	<tr>
		<td>Segment Selector</td>
		<td>16 bits</td>
		<td>Specifies the code segment to be accessed.</td>
	</tr>
	<tr>
		<td>P (Present)</td>
		<td>1 bit</td>
		<td>Specifies if the call gate is valid or not (1 = valid, 0 = invalid).</td>
	</tr>
	<tr>
		<td>DPL</td>
		<td>2 bits</td>
	</tr>
	<tr>
		<td>Type</td>
		<td>4 bits</td>
		<td>Sets the call gate bit and specifies the size of values to be pushed onto the target stack.</td>
	</tr>
	<tr>
		<td>Param. Count</td>
		<td>5 bits</td>
		<td>Specifies the number of parameters to copy from the calling procedure stack to the new stack if a stack switch occurs.</td>
	</tr>
</table>

- IA-32e mode call gate descriptors are as follows:
```
 127                                                               107  106     104  103              96
+----------------------------------------------------------------------+------------+--------------------+
|                                  Reserved                            |    00000   |      Reserved      |
+----------------------------------------------------------------------+------------+--------------------+
 95                                                                                                   64
+--------------------------------------------------------------------------------------------------------+
|                                           Entry Point in Segment 63:32                                 |
+--------------------------------------------------------------------------------------------------------+
 63                                                     48  47  46  45  44  43   40  39               32
+----------------------------------------------------------+---+-------+---+--------+--------------------+
|                Entry Point in Segment 31:16              | P |  DPL  | 0 |  1100  |      00000000      |
+----------------------------------------------------------+---+-------+---+--------+--------------------+
 31                                                     16  15                                        00
+----------------------------------------------------------+---------------------------------------------+
|                     Segment Selector                     |         Entry Point in Segment 15:00        |
+----------------------------------------------------------+---------------------------------------------+
```


### Segment Descriptor Tables
- A segment descriptor table is an array of segment descriptors whose length is up to 8192 8-byte descriptors.
- There are two kinds of descriptor tables:
	- The global descriptor table (GDT)
	- The local descriptor table (LDT)
- Each system must have one GDT defined, which may be used for all programs and tasks in the system. Optionally, one or more LDTs can be defined.
- GDT
	- The GDT is not a segment itself, instead, it is a data structure in linear address space.
	- The base linear address and limit of the GDT must be loaded into the GDTR.
		- The base address should be aligned on an eight-byte boundary.
		- The limit value is expressed in bytes. Bacuase the segment descriptors are always 8 bytes long, the limit should be one less than an integral multiple of eight (that is, 8N - 1).
	- The first descriptor is a null descriptor and is not used by the processor.
		- A segment selector to this null descriptor does not generate an exception when loaded into a data-segment register (DS, ES, FS, or GS), but it always generates a general-protection exception (#GP) when an attempt is made to acecss memory using the descriptor.
- LDT
	- The LDT is located in a system segment of the LDT type.
	- The GDT must contain a segment descriptor for the LDT segment.



## Paging (Under Construction)
- If paging is not used, the linear address space is mapped directly into the physical address space.
- Paging supports a virtual memory environment where a large linear address space is simulated with a small amount of physical memory and some disk storage.
- When using paging, each segment is divided into pages (typically 4 KiB each in size), which are stored either in physical memory or on the disk.
- How to convert address
	- The operating system maintains a page directory and a set of page tables to keep track of the pages.
	- When a program attemps to access an address location in the linear address space, the processor uses the page directory and page tables to translate the linear address into a physical address and then performs the requested



## Real Mode


## Protected Mode


## IA-32e Mode


