---
layout: post
title: Multitasking
---


## Task Structure
- A task is made up of two parts: a task execution space and a task-state segment (TSS).
	- Task execution space
		- The task execution space consists of a code segment, a stack segment, and one or more data segments.
		- If an operating system uses the processor's privilege-level protection mechanism, the task execution space also provides a separate stack for each privilege level.
	- TSS
		- The TSS specifies the segments that make up the task execution space and provides a storage place for task state information.
		- A task is identified by the segment selector for its TSS.
		- When a task is loaded into the processor for execution, the segment selector, base address, limit and segment descriptor attributes for the TSS are loaded into the task register (TR).
		- If paging is enabled, the base address of the page directory used by the task is loaded into CR3.


### Task State
- The following items define the state of the currently executing task:
	- The task's current execution space, defined by the segment selectors in the segment registers (CS, DS, SS, ES, FS and GS).
	- The state of the general purpose registers.
	- The state of the EFLAGS register.
	- The state of the EIP register.
	- The state of the CR3.
	- The state of the TR.
	- The state of the LDTR.
	- The I/O map base address and I/O map.
	- Stack pointers to the privilege 0, 1, and 2 stacks.
	- Link to previously executed task.
	- The state of the shadow stack pointer.
- Prior to dispatching a task, all of these items are contained in the task's TSS, except the state of the TR.


### Executing a Task
- Software or the processor can dispatch a task for execution in one of the following ways:
	- A explicit call to a task with the CALL instruction.
	- A explicit jump to a task with the JMP instruction.
	- An implicit call (by the processor) to an interrupt handler task.
	- An implicit call to an exception handler task.
	- A return (initiated with an IRET instruction) when the NT flag in the EFLAGS register is set.
- All of these methods for dispatching a task identify the task to be dispatched with a segment selector that points to a task gate or the TSS for the task.
- When a task is dispatched for execution, a task switch occurs between the currently running task and the dispatched task.
	- During a task switch, the execution environment of the currently executing task (called context) is saved in its TSS and execution of the task is suspended.
	- The context for the dispatched task is then loaded into the processor and execution of that task begins with the instruction pointed to by the newly loaded EIP register.
	- If the calling task called the called task, the TSS segment selector for the calling task is stored in the TSS of the called task to provide a link back to the calling task.



## Task Management Data Structures
- The processor defines five data structures for handling task related activities:
	- Task state segment (TSS)
	- Task gate descriptor
	- TSS descriptor
	- Task register
	- NT flag in the EFLAGS register
- When operating in protected mode, a TSS and TSS descriptor must be created for at least one task, and the segment selector for the TSS must be loaded into the task register.


### Task State Segment (TSS)
#### 32-bit TSS
```
 31                  16 15                  00
+----------------------+----------------------+
|          SSP (Shadow Stack Pointer)         | 104
+----------------------+--------------------+-+
| I/O Map Base Address |      Reserved      |T| 100
+----------------------+--------------------+-+
|       Reserved       | LDT Segment Selector | 96
+----------------------+----------------------+
|       Reserved       |          GS          | 92
+----------------------+----------------------+
|       Reserved       |          FS          | 88
+----------------------+----------------------+
|       Reserved       |          DS          | 84
+----------------------+----------------------+
|       Reserved       |          SS          | 80
+----------------------+----------------------+
|       Reserved       |          CS          | 76
+----------------------+----------------------+
|       Reserved       |          ES          | 72
+----------------------+----------------------+
|                     EDI                     | 68
+---------------------------------------------+
|                     ESI                     | 64
+---------------------------------------------+
|                     EBP                     | 60
+---------------------------------------------+
|                     ESP                     | 56
+---------------------------------------------+
|                     EBX                     | 52
+---------------------------------------------+
|                     EDX                     | 48
+---------------------------------------------+
|                     ECX                     | 44
+---------------------------------------------+
|                     EAX                     | 40
+---------------------------------------------+
|                    EFLAGS                   | 36
+---------------------------------------------+
|                     EIP                     | 32
+---------------------------------------------+
|                     CR3                     | 28
+----------------------+----------------------+
|       Reserved       |         SS2          | 24
+----------------------+----------------------+
|                     ESP2                    | 20
+----------------------+----------------------+
|       Reserved       |         SS1          | 16
+----------------------+----------------------+
|                     ESP1                    | 12
+----------------------+----------------------+
|       Reserved       |         SS0          | 8
+----------------------+----------------------+
|                     ESP0                    | 4
+----------------------+----------------------+
|       Reserved       |  Previous Task Link  | 0
+----------------------+----------------------+
```

#### 16-bit TSS
- The 32-bit IA-32 processors also recognize a 16-bit TSS format like the one used in Intel 286 processors.

```
 15                  00
+----------------------+
| LDT Segment Selector | 42
+----------------------+
|          DS          | 40
+----------------------+
|          SS          | 38
+----------------------+
|          CS          | 36
+----------------------+
|          ES          | 34
+----------------------+
|          DI          | 32
+----------------------+
|          SI          | 30
+----------------------+
|          BP          | 28
+----------------------+
|          SP          | 26
+----------------------+
|          BX          | 24
+----------------------+
|          DX          | 22
+----------------------+
|          CX          | 20
+----------------------+
|          AX          | 18
+----------------------+
|         FLAG         | 16
+----------------------+
|          IP          | 14
+----------------------+
|          SS2         | 12
+----------------------+
|          SP2         | 10
+----------------------+
|          SS1         | 8
+----------------------+
|          SP1         | 6
+----------------------+
|          SS0         | 4
+----------------------+
|          SP0         | 2
+----------------------+
|  Previous Task Link  | 0
+----------------------+
```

#### 64-bit TSS
- In 64-bit mode, task structure and task state are similar to those in protected mode.
- However, the task switching mechanism available in protected mode is not supported in 64-bit mode.
- Task management and switching must be performed by software.

```
 31                  16 15                  00
+----------------------+----------------------+
| I/O Map Base Address |       Reserved       | 100
+----------------------+----------------------+
|                   Reserved                  | 96
+---------------------------------------------+
|                   Reserved                  | 92
+---------------------------------------------+
|             IST7 (upper 32 bits)            | 88
+---------------------------------------------+
|             IST7 (lower 32 bits)            | 84
+---------------------------------------------+
|             IST6 (upper 32 bits)            | 80
+---------------------------------------------+
|             IST6 (lower 32 bits)            | 76
+---------------------------------------------+
|             IST5 (upper 32 bits)            | 72
+---------------------------------------------+
|             IST5 (lower 32 bits)            | 68
+---------------------------------------------+
|             IST4 (upper 32 bits)            | 64
+---------------------------------------------+
|             IST4 (lower 32 bits)            | 60
+---------------------------------------------+
|             IST3 (upper 32 bits)            | 56
+---------------------------------------------+
|             IST3 (lower 32 bits)            | 52
+---------------------------------------------+
|             IST2 (upper 32 bits)            | 48
+---------------------------------------------+
|             IST2 (lower 32 bits)            | 44
+---------------------------------------------+
|             IST1 (upper 32 bits)            | 40
+---------------------------------------------+
|             IST1 (lower 32 bits)            | 36
+---------------------------------------------+
|                   Reserved                  | 32
+---------------------------------------------+
|                   Reserved                  | 28
+---------------------------------------------+
|             RSP2 (upper 32 bits)            | 24
+---------------------------------------------+
|             RSP2 (lower 32 bits)            | 20
+---------------------------------------------+
|             RSP1 (upper 32 bits)            | 16
+---------------------------------------------+
|             RSP1 (lower 32 bits)            | 12
+---------------------------------------------+
|             RSP0 (upper 32 bits)            | 8
+---------------------------------------------+
|             RSP0 (lower 32 bits)            | 4
+---------------------------------------------+
|                   Reserved                  | 0
+---------------------------------------------+
```


### TSS Descriptor
- The TSS is defined by a segment descriptor.
- TSS descriptors may only be placed in the GDT; they cannot be placed in an LDT or the IDT.
- The busy flag (B)
	- It indicates whether the task is busy.
	- A busy task is currently running or suspended.
	- Tasks are not recursive.
	- The processor uses the busy flag to detect an attempt to call a task whose execution has been interrupted.
- The granurality flag (G)
	- When the G flag is 0 in a TSS descriptor for a 32-bit TSS.

#### 32-bit TSS Descriptor
```
 63               56  55  54  53  52  51                48  47  46  45  44  43   40  39               32
+--------------------+---+---+---+---+---------------------+---+-------+---+--------+--------------------+
| Base Address 31:24 | G | 0 | 0 |AVL| Segment Limit 19:16 | P |  DPL  | 0 |  10B1  | Base Address 23:16 |
+--------------------+---+---+---+---+---------------------+---+-------+---+--------+--------------------+
 31                                                     16  15                                        00
+----------------------------------------------------------+---------------------------------------------+
|                    Base Address 15:00                    |              Segment Limit 15:00            |
+----------------------------------------------------------+---------------------------------------------+
```

#### 64-bit TSS Descriptor
```
 127                                                               107  106     104  103              96
+----------------------------------------------------------------------+------------+--------------------+
|                                  Reserved                            |    00000   |      Reserved      |
+----------------------------------------------------------------------+------------+--------------------+
 95                                                                                                   64
+--------------------------------------------------------------------------------------------------------+
|                                                  Base Address 63:32                                    |
+--------------------------------------------------------------------------------------------------------+
 63               56  55  54  53  52  51                48  47  46  45  44  43   40  39               32
+--------------------+---+---+---+---+---------------------+---+-------+---+--------+--------------------+
| Base Address 31:24 | G | 0 | 0 |AVL| Segment Limit 19:16 | P |  DPL  | 0 |  10B1  | Base Address 23:16 |
+--------------------+---+---+---+---+---------------------+---+-------+---+--------+--------------------+
 31                                                     16  15                                        00
+----------------------------------------------------------+---------------------------------------------+
|                    Base Address 15:00                    |              Segment Limit 15:00            |
+----------------------------------------------------------+---------------------------------------------+
```


### Task Regsiter
- The task register holds the 16-bit segment selector (visible part) and the entire segment descriptor (invisible part) for the TSS of the current task.
- This information is copied from the TSS descriptor in the GDT for the current task.
- The LTR (load task register) instruction
	- It loads a segment selector into the task register that points to a TSS descriptor in the GT.
	- LTR is a privileged instruction that may be executed only when the CPL is 0.
	- It's used during system initialization to put an initial value in the task register.
	- Afterwards, the contents of the task register are changed implicitly when a task switch occurs.


### Task-Gate Descriptor
- A task-gate descriptor provides an indirect, protected reference to a task.
- It can be placed in the GDT, an LDT, or the IDT.
- The DPL controls access to the TSS descriptor during a task switch.
	- When a program makes a call or jump to a task through a task gate, the CPL and the RPL of the gate selector pointing to the task gate must be less thatn or eqal to the DPL of the task-gate descriptor.
	- Note that when a task gate is used, the DPL of the destination TSS descriptor is not used.

```
 63                                                     48  47  46  45  44  43   40  39               32
+----------------------------------------------------------+---+-------+---+--------+--------------------+
|                         Reserved                         | P |  DPL  | 0 |  0101  |      Reserved      |
+----------------------------------------------------------+---+-------+---+--------+--------------------+
 31                                                     16  15                                        00
+----------------------------------------------------------+---------------------------------------------+
|                    TSS Segment Selector                  |                   Reserved                  |
+----------------------------------------------------------+---------------------------------------------+
```



## Task Switching
- The processor transfers execution to another task in one of four cases:
	- The current program executes a JMP or CALL instruction to a TSS descriptor in the GDT.
	- The current program executes a JMP or CALL instruction to a task-gate descriptor in the GDT or the current LDT.
	- An interrupt or exception vector points to a task-gate descriptor in the IDT.
	- The current task executes an IRET when the NT flag in the EFLAGS register is set.
- The state of the currently executing task is always saved when a successful task switch occurs. If the task is resumed, execution starts with the instructioon pointed to by the saved EIP, and the registers are restored to the values they held when the task was suspended.
