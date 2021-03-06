---
layout: post
title: Architectures & Modes
---


## Architectures & Modes
- IA-32 arthictecture (i386 / x86)
	- Real Mode (16-bit mode)
		- Provides the programming environment of the Intel 8086 processor, with a few extensions.
	- Protected Mode (32-bit mode)
		- Provides a rich set of architectual features, flexibility, high performance and backward compatibility to existing softwares.
	- Virtual 8086 Mode
		- Allows the processor to execute 8086 software in a protected, multitasking environment.
	- System Management Mode (SMM)
		- Provides an operating system with a transparent mechanism for implementing power management and OEM differentiation features.
- Intel 64 architecture (x86-64 / x64 / AMD64 / IA-32e / EM64T)
	- IA-32e Mode
		- Supports two sub-modes:
			- 64-bit Mode
				- Supports 64-bit OS and 64-bit applications.
			- Compatibility Mode
				- Allows most legacy protected mode applications to run unchanged.



## Mode Switching
### Switching to Protected Mode
- Before switching to protected mode from real mode, a minimum set of system data structures and code modules must be loaded into memory.
- Protected mode is entered by executing a MOV CR0 instruction that sets the PE flag in the CR0. Execution in protected mode begins with a CPL of 0.
- Steps
	1. Disable A20 gate.
	1. Disable interrupts with a CLI instruction.
	1. Execute the LGDT instruction to load the GDTR with the base address and limit of the temporary GDT.
	1. Execute a MOV CR0 instruction that sets the PE flag (and optionally the PG flag) in CR0 to switch to protected mode.
	1. Immedaitely following the MOV CR0 instruction, execute a JMP instruction to flush the pipeline.
	1. Reload segment registers.
		- Reload segment registers DS, SS, ES, FS, and GS.
		- Perform a far JMP or far CALL instruction to reset CS.
	1. Execute the LGDT instruction to loda the GDTR with the base address and limit of protected mode GDT.
	1. Execute the LIDT instruction to load the IDTR with the base address and limit of protected mode IDT.
	1. Execute the STI instruction to enable maskable hardware interrupts.
	1. If a local descriptor table is going to be used, execute the LLDT instruction to load the segment selector for the LDT in the LDTR.
	1. Execute the LTR instruction to load a segment selector for a TSS descriptor into the task register.
		- This instruction marks the TSS descriptor as busy, but does not perform a task switch.
		- The segment selector for the TSS must be loaded before software performs its first task switch in protected mode, because a task switch copies the current task state into the TSS.
		- After the first LTR instruction has been executed, further operations on the task register are performed by task switching.


### Switching to IA-32e Mode
- Steps
	1. Disable paging by setting CR0.PG = 0.
	1. Enable physical address extension (PAE) by setting CR4.PAR = 1.
	1. Load CR3 with the physical base address of the level 4 page map table (PML4) or level 5 page map table (PML5).
	1. Enable IA-32e mode by setting IA32_EFER.LME = 1.
	1. Enable paging by settings CR0.PG = 1. This causes the processor to set the IA32_EFER.LMA = 1.



## System Registers
### CR0
```
 31 30 29 28 27 26 25 24 23 22 21 20 19 18 17 16 15 14 13 12 11 10 09 08 07 06 05 04 03 02 01 00
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|PG|CD|NW|                             |AM|  |WP|                             |NW|ET|TS|EM|MP|PE|
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

<table>
	<tr>
		<th>Bit</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>31 (PG)</td>
		<td>
			Paging<br>
			Enables paging when set; disables paging when clear.
		</td>
	</tr>
	<tr>
		<td>00 (PE)</td>
		<td>
			Protection Enable<br>
			Enables protected mode when set; enables read mode when clear.
		</td>
	</tr>
</table>


### Extended Feature Enable Register (IA32_EFER)
- The IA32_EFER provides several fields related to IA-32e mode enabling and operation.

```
 63  12 11  10  09  08  07  01 00
+--//--+---+---+---+---+--//--+---+
|      |NXE|LMA|   |LME|      |SCE|
+--//--+---+---+---+---+--//--+---+
```

<table>
	<tr>
		<th>Bit</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>10 (LMA)</td>
		<td>
			IA-32e Mode Active (R)<br>
			Indicates IA-32e mode is active.
		</td>
	</tr>
	<tr>
		<td>8 (LME)</td>
		<td>
			IA-32e Mode Enable (R/W)<br>
			Enables IA-32e mode operation.
		</td>
	</tr>
</table>
