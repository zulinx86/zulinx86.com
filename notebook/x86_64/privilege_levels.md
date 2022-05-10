---
layout: post
title: Privilege Levels
---


## Privilege Levels
- The processor's segment-protection mechanism recognizes 4 privilege levels, numbered from 0 to 3.
	- Levels
		- Level 0: Operating system kernel
		- Level 1, 2: Operating system services
		- Level 3: Applications
	- The greater numbers mean lesser privileges.
	- The processor uses privilege levels to prevent a program operating at a lesser privilege level from accessing a segment with a greater privilege, except under controlled situations.
	- When the processor detects a privilege level violation, it generates a general-protection exception (#GP).
- Types of privilege levels:
	- Current Privilege Level (CPL)
		- The CPL is the privilege level of the currently executing program.
		- It is stored in bits 0 and 1 of the CS and SS segment registers.
		- Normally, the CPL is equal to the privilege level of the code segment from which instructions are being fetched.
		- The processor changes the CPL when program control is transferred to a code segment with a different privilege level.
		- Conforming code segments can be accessed from any privilege level that is eqaul to or less privileged than the DPL of the conforming code segment.
		- Also, the CPL is not changed when the processor accesses a conforming code segment that has a different privilege level than the CPL.
	- Descriptor Privilege Level (DPL)
		- The DPL is the privilege level of a segment or gate.
		- It is stored in the DPL field of the segment or gate descriptor.
		- When the currently executing code segment attempts to access a segment or gate, the DPL is compared to the CPL and RPL of the segment or gate selector.
		- The DPL is interpreted differently, depending on the type of segment or gate being accessed:
			- Data segment
				- The DPL indicates the numerically highest privilege level that a program can have to be allowed to access the segment.
				- For example, if the DPL is 1, only programs running at a CPL of 0 or 1 can access the segment.
			- Non-conforming code segment (without using a call gate)
				- The DPL indicates the privilege level that a program must be at to access the segment.
				- For example, if the DPL is 0, only programs running at a CPL of 0 can access the segment.
			- Call gate
				- The DPL indicates the numerically highest privilege level that the current program can be at and still be able to access the call gate.
				- This is the same access rule as for a data segment.
			- Conforming code segment and non-conforming code segment accessed through a call gate
				- The DPL indicates the numerically highest privilege level that a program can have to be allowed to access the segment.
				- For example, if the DPL is 2, programming rnning at a CPL of 0 or 1 cannot access the segment.
			- TSS
				- Ths DPL indicates the numerically highest privilege level that the current program can be at and still be able to access the TSS.
				- This is the same access rule as for a data segment.
	- Requested Privilege Level (RPL)
		- The RPL is an override privielge level that is assigned to segment selectors.
		- It is stored in bits 0 and 1 of the segment selector.
		- The processor checks the RPL along with the CPL to determine if access to a segment is allowed.
		- If the RPL of a segment seletor is numerically greater than the CPL, the RPL overrides the CPL, and vice versa.



## Privilege Level Checks
- Privilege levels are checked when the segment selector of a segment descriptor is loaded into a segment register.
- The checks used for data access differ from those used for transfer of program control among code segments.


### Accessing Data Segments
- To access operands in a data segment, the segment selector for the data segment must be loaded into the data segment registers (DS, ES, FS, or GS) or into the stack-segment register (SS).
- The DPL should be nuemrically greater than or equal to both the CPL and the RPL (**DPL >= CPL and DPL >= RPL**). Otherwise, a general-protection fault is generated and the segment register is not loaded.


### Accessing Data in Code Segments
- The following methods of accessing data in code segments are possible:
	1. Load a data-segment register with a segment selector for a non-conforming, readable, code segment.
		- The same rules for accessing data segments apply to this.
	1. Load a data-segment register with a segment selector for a conforming, readable, code segment.
		- This is always valid because the privilege level of a conforming code segment is effectively the same as the CPL, regardless of its DPL.
	1. Use a code-segment override prefix (CS) to read a readable, code segment whose selector is already loaded in the CS regsiter.
		- This is always valid because the DPL of the code segment selected by the CS register is the same as the CPL.


### Loading the SS Register
- The CPL, the RPL of the stack-segment selector, and the DPL of the stack-segment descriptor must be the same (**CPL == RPL == DPL**).
- If the RPL and DPL are not equal to the CPL, a general-protection exception (#GP) is generated.


### Transferring Program Control Between Code Segments
- To transfer program control from one code segment to another, the segment selector for the destination code segment must be loaded into the code-segment register (CS).
- Program control transfers are carried out with the JMP, CALL, RET, SYSENTER, SYSEXIT, SYSCALL, SYSRET, INT n, and IRET instructions, as well as by the exception and interrupt mechanisms.

#### Direct Calls or Jumps to Code Segments
- The near forms of the JMP, CALL, and RET instructions transfer control within the current code segment, so privlege-level checks are not performed.
- The far forms of the JMP, CALL, and RET instructions transfer control to other code segments, so the processor does perform privilege-level checks.
- When transferring program control to another code segment without going through a call gate, the processor examines four kinds of privilege level and type information.
	- The CPL (the privilege level of the calling code segment)
	- The DPL of the segment descriptor for the destination code segment
	- The RPL of the segment selector of the destination code segment
	- The conforming flag in the segment descriptor for the destination code segment
- Accessing Non-conforming Code Segments
	- The CPL of the calling procedure must be eqaul to the DPL of the destination code segment; otherwise, the processor generates a general-protection exception (#GP).
- Accessing Conforming Code Segments
	- The CPL of the calling procedure must be numerically equal to or greater than the DPL of the destination code segment; the processor generates a general-protection exception (#GP) only if the CPL is less than the DPL.
	- The segment selector's RPL for the destination code segment is not checked if the segment is a conforming code segment.
	- The CPL does not change before and after control transfer, even if the DPL of the destination code segment is less than the CPL. This situation is the only one where the CPL may be different from the DPL of the current code segment. Also, since the CPL does not change, no stack switch occurs.

#### Calls or Jumps through Call Gates
- Call gates facilitate controlled transfers of program control between different privilege levels.
- To access a call gate, a far pointer to the gate is provided as a target operand in a CALL or JMP instruction.
	- The segment selector from this far pointer identifies the call gate; the offset from this far pointer is required, but not used or checked by the processor (the offset can be set to any value).
	- linear address of the procedure entry point = base address from the code-segment descriptor + offset from the call gate.

- Four different privilege levels are used to check the validity:
	- The CPL
	- The RPL of the call gate's selector
	- The DPL of the call gate descriptor
	- The DPL of the segment descriptor of the destination code segment
- The privilege checking rules are different depending on whether the control transfer was initiated with a CALL or a JMP instruction:
	- CALL
		- CPL <= call gate DPL
		- RPL <= call gate DPL
		- Destination (conforming and non-conforming) code segment DPL <= CPL
	- JMP
		- CPL <= call gate DPL
		- RPL <= call gate DPL
		- Destination conforming code segment DPL <= CPL
		- Destination non-conforming code segment DPL = CPL
- If a call is made to a more privileged non-conforming destination code segment, the CPL is lowered to the DPL of the destination code segment and a stack switch occurs.
- If a call or jump is made to a more privileged conforming destination code segment, the CPL is not changed and no stack switch occurs.

#### Returning from a Called Procedure
- The RET instruction can be used to perform a near return, a far return at the same privilege level, and a far return to a different privilege level.
	- A near return:
		- It only transfers program control within the current code segment; therefore, the processor performs only a limit check.
	- A far return at the same privilege level
		- The processor pops both a segment selector and a return instruction pointer from the stack.
		- The processor performs privilege checks to detect situations where the current procedure might have altered the pointer or failed to maintain the stack properly.
	- A far return to a different privilege level
		- It is only allowed then returning to a less privilege level.
		- The processor uses the RPL from the CS saved for the calling procedure to determine if a return to a numerically higher privilege level is required.
- This instruction is intended to execute returns from procedures that were called with a CALL instruction. It does not support returns from a JMP instruction.

#### Task Switching through TSS or Task Gates
- Checks that the current (old) task is allowed to switch to the new task.
	- Data-access privilege rules apply to JMP and CALL instructions. The CPL of the current (old) task and the RPL of the segment selector for the new task must be less than or eqaul to the DPL of the TSS descriptor or task gate being referenced.
	- Exceptions, interrupts, and the IRET and INT1 instructions are permitted to switch tasks regardless of the DPL of the destination task-gate or TSS descriptor.	
	- For interrupts generated by the INT n, INT3, and INTO instructions, the DPL is checked and a general-protection exception (#GP) results if it is less than the CPL.

#### SYSENTER and SYSEXIT
- The SYSENTER and SYSEXIT instructions were introduced into the IA-32 architecture in the Pentium II processors for the purpose of providing a fast (low overhead) mechanism for calling operating system procedures.
- SYSENTER is intended for use by user code running at privilege level 3 to access operating system procedures running at privilege level 0.
- SYSEXIT can be executed from privilege level 3, 2, 1, or 0; SYSEXIT can only be executed from privilege level 0.
- The SYSENTER and SYSEXIT instructions are companion instructions, but they do not constitute a call/return pair. This is because SYSENTER does not save any state information for use by SYSEXIT on a return.

#### SYSCALL and SYSRET
- The instructions, along with SYSENTER and SYSEXIT, are suited for IA-32e mode operation.
- SYSCALL and SYSRET are not supported in compatibility mode.
- SYSCALL is intended for use by user code running at privilege level 3 to access operating system procedures running at privilege level 0.
- SYSRET is intended for use by privilege level 0 operating system procedures for fast returns to privielge level 3 user code.