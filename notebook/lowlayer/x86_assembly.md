---
layout: post
title: x86 Assembly
---

## Calling Convention
- A calling convetion is an implementation-level scheme for:
	- Arguments: How subroutines receive arguments from their caller.
		- When using registers, which registers are used.
		- When using a stack, whether to stack the first argument first or last.
	- Return value: How subroutines return its result.
		- Which register is used to store the result.
	- Retention of registers: Which registers should be retained before and after calling subroutines.


### System V ABI
- Used by the major Unix operating systems such as Linux, the BSD systems, and many others.

#### Intel386 (i386)
- Arguments
	- All arguments are pushed onto the stack.
	- The first argument is pushed lastly.
- Return value
	- The return value is stored in EAX.
	- If it is 64 bit value, the upper 32 bit value is stored in EDX.
- Retention of registers
	- Registers should be retained: EBX, ESI, EDI, EBP, ESP
	- Registers don't have to be retained: EAX, ECX, EDX
- How to process
	1. Push arguments onto the stack
	1. Execute `call` instruction
		- The address of the next instruction is pushed onto the stack.
		- EIP is set to the operand address of `call` instruction.
	1. Push EBP to the stack (`push ebp`).
	1. Move ESP to EBP (`mov ebp,esp`).
	1. Execute the main part of the function
	1. Restore ESP (`mov esp,ebp`).
	1. Pop from the stack to EBP (`pop ebp`).
	1. Execute `ret` instruction.
		- Pop from the stack to EIP.
- Stack frame
	```
	┌───────────┬─────────────────────────────┐
	│    0(%esp)│                             │
	├───────────┼─────────────────────────────┤
	│           │            ...              │
	├───────────┼─────────────────────────────┤
	│   -4(%ebp)│       local variable        │
	├───────────┼─────────────────────────────┤
	│    0(%ebp)│     previous %ebp value     │
	├───────────┼─────────────────────────────┤
	│    4(%ebp)│       return address        │
	├───────────┼─────────────────────────────┤
	│    8(%ebp)│ 0-th 4-byte memory argument │
	├───────────┼─────────────────────────────┤
	│           │            ...              │
	├───────────┼─────────────────────────────┤
	│ 4n+8(%ebp)│ n-th 4-byte memory argument │
	└───────────┴─-───────────────────────────┘
	```

#### AMD64 (x86-64)
- Arguments
	- Integer / Pointer: RDI, RSI, RDX, RCX, R8, R9
	- Floating-point: XMM0, XXM1, XMM2, XMM3, XMM4, XMM5, XMM6, XMM7
	- If the registers are not enough, it uses the stack.
	- In system call, R10 is used instead of RCX.
- Return value
	- The return value is stored in RAX.
	- If it is 128 bit value, the upper 64 bit value is stored in RDX.
- Retention of registers
	- Registers should be retained: RBX, RSP, RBP, R12, R13, R14, R15
	- Registers don't have to be retained: RAX, RDI, RSI, RDX, RCX, R8, R9, R10, R11
- How to process
	1. Set arguments to the registers and the stack
	1. Execute `call` instruction
		- The address of the next instruction is pushed onto the stack.
		- RIP is set to the operand address of `call` instruction.
	1. Push RBP to the stack (`push rbp`).
	1. Move RSP to RBP (`mov rbp,rsp`).
	1. Execute the main part of the function
	1. Restore RSP (`mov rsp,rbp`).
	1. Pop from the stack to RBP (`pop rbp`).
	1. Execute `ret` instruction.
		- Pop from the stack to RIP.
- Stack frame
	```
	┌─────────────────┬─────────────────────────────┐
	│         0(%rsp) │                             │
	├─────────────────┼─────────────────────────────┤
	│                 │            ...              │
	├─────────────────┼─────────────────────────────┤
	│        -8(%rbp) │       local variable        │
	├─────────────────┼─────────────────────────────┤
	│         0(%rbp) │     previous %rbp value     │
	├─────────────────┼─────────────────────────────┤
	│         8(%rbp) │       return address        │
	├─────────────────┼─────────────────────────────┤
	│        16(%rbp) │  6th 8-byte memory argument │
	├─────────────────┼─────────────────────────────┤
	│                 │            ...              │
	├─────────────────┼─────────────────────────────┤
	│ 8(n-6)+16(%rbp) │ n-th 8-byte memory argument │
	└─────────────────┴─────────────────────────────┘

	1st argument: RDI
	2nd argument: RSI
	3rd argument: RDX
	4th argument: RCX
	5th argument: R8
	6th argument: R9
	```



## Intel syntax vs. AT&T syntax

|                   | Intel                     | AT&T                       |
| ----------------- | ------------------------- | -------------------------- |
| Immediates        | no prefix                 | prefixed with `$`          |
| Registers         | no prefix                 | prefixed with `%`          |
| Operands Order    | `instr dst, src`          | `instr src, dst`           |
| Mnemonic          | no suffix                 | suffixed with operand size |
| Effective Address | `[base+index*scale+disp]` | `disp(base,index,scale)`   |



## Netwide Assember (NASM)
- What is NASM?
	- It is an assembler and disassembler for the Intel x86 architecture (16-bit, 32-bit and 64-bit).
	- It can output several binary formats, including COFF, OMF, a.out, ELF, Mach-O abd binary file.
	- It uses Intel syntax intead of AT&T syntax.
- `nasm` command
	- `-f <oformat>`: speciy the output format
		- `bin`: raw binary
		- `aout`: Linux a.out
		- `aoutb`: NetBSD/FreeBSD a.out
		- `coff`: COFF (i386)
		- `elf32`: ELF32 (i386)
		- `elf64`: ELF64 (x86-64)
		- `elf`: legacy alias for `elf32`
	- `-o <filename>`: specify the output file name
	- `-l <filename>`: generate a listing file
- Pseudo-instructions
	- `dx`: declare initialized data
		- `db`: declare data in byte
		- `dw`: declare data in word
		- `dd`: declare data in double word
		- `dq`: declare data in quad word
	- `resx`: declare uninitialized data
		- `resb`: reserve in byte
		- `resw`: reserve in word
		- `resd`: reserve in double word
		- `resq`: reserve in quad word
	- `equ`: define constants
	- `times`: repeat instructions or data
- Constants
	- Numeric constants
		- Binary: `0bXXXX`, `XXXXb`
		- Octal: `0oXXXX`, `XXXXo`
		- Decimal: `XXXX`, `0dXXXX`, `XXXXd`
		- Hexadecimal: `0xXXXX`, `XXXXh`
	- String constants
		- `db 'hello'`
		- `db 'h','e','l','l','o'`
- Expressions
	- `$`: the assembly position at the beginning of the line containing the expression.
	- `$$`: the beginning of the current section. (`$-$$`: how far into the section you are)
- Directives
	- `bits xx`: specify whether NASM should generate code designed to run on a processor operating in 16, 32 and 64 bit
		- `bin` output format defaults to 16 bit mode. The most likely reason for using this `bits` directive is to write 32 or 64 bit code in a flat binary file.
		- `bits` directive has an exactly equivalent primitive form `[bits 16]`, `[bits 32]` and `[bits 64]`.
	- `usexx`: aliases for `bits` (`use16` for `bits 16`, `use32` for `bits 32`)
	- `global`: export symbols to other modules
	- `extern`: import symbols from other modules
		- It is used to declare a symbol which is not defined anywhere in the module being assembled
	- `section` or `segment`: changes which section of the output file the code will be assembled into.
		- `section` directive is unusual in that its user-level form functions differently from its primitive form.
		- The primitive form `[section xyz]` simply switches the current target section to the one given.
		- The user-level form `section xyz` first defines the single-line macro `__SECT__` to be the primitive `[section]` directive and then issue it.
		- `section xyz` expands to the following two lines
			```
			%define __SECT__ [section xyz]
			        [section xyz]
			```
- Local labels
	- A label beginning with a single period `.` is treated as a local label and it is associated with the previous non-local label.
		```
		label1:
			; some code
		.loop:			; associated with label1
			; some more code
			jne .loop	; jump to label1.loop
			ret

		label2:
			; some code
		.loop:			; associated with label2
			; some more code
			jne .loop	; jump to label2.loop

		label3:
			; some code
			jmp label1.loop
		```
- Output formats
	- `bin`: flat-form binary output
		- 16-bit mode is default in NASM. In order to write 32 or 64 bit code, you need to explicitly issue the `bits 32` or `bits 64` directive.
		- `org`: specify the origin address which NASM will assume the program begins at when it is loaded into memory.
		- `section <name> align=<byte>`: switch to the specified section aligned on a specified byte boundary.
	- `coff`: common object file format
	- `macho32` / `macho64`: Mach-O file format
	- `elf32` / `elf64`: executable and linkable format
	- `aout`: Linux a.out object file
- x86 instructions
	- operand specifications
		- registers: `reg8` / `reg16` / `reg32`
		- immediate: `imm8` / `imm16` / `imm32`
		- memory: `mem8` / `mem16` / `mem32` / `mem64`
		- register or memory: `r/m8` / `r/m16` / `r/m32`
	- opcode descriptions
		- `ib` / `iw` / `id`: immediate value, and that this is encoded as a byte, little-endian word or little-endian double word.
		- `rb` / `rw` / `rd`: immediate value, and that the difference between this value and the address of end of the instruciton is to be encoded as a byte, word or doubleword.
	- register values
		- 8 bit general registers: AL, CL, DL, BL, AH, CH, DH, BH
		- 16 bit general registers: AX, CX, DX, BX, SP, BP, SI, DI
		- 32 bit general registers: EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI
		- segment registers: ES, CS, SS, DS, FS, GS
		- control registers: CR0, CR2, CR3, CR4


### Examples
#### Hello World
```asm
global _start

bits 64
section .text
_start:
        ; write(1, message, 14)
        mov rax, 1      ; system call number is 1 (write)
        mov rdi, 1      ; file handler is 1 (stdout)
        mov rsi, msg    ; address of string to output
        mov rdx, len    ; number of bytes
        syscall         ; invoke system call

        ; exit(0)
        mov rax, 60     ; system call number is 60 (exit)
        xor rdi, rdi    ; return code is 0
        syscall         ; invoke system call

section .data
msg:    db "Hello, world!",10
len     equ $ - msg
```
```sh
$ nasm -f elf64 -o hello.o hello.asm
$ gcc -nostdlib hello.o
$ ./a.out
Hello, world!
```



## GNU Assembler (GAS)
- What is GNU Assember?
	- It is the assembler developed by the GNU project.
	- It is the default backend of GCC.
	- It is a part of the GNU Binutils package.
	- Its executable is named `as`, the standard name for a Unix assemlber.
	- It is cross-platform, and both runs on and assembles for a number of different computer architectures.
- Syntax
	- Intel syntax: `.intel_syntax noprefix`
	- AT&T syntax (default): `.att_syntax`


### Examples
#### Hello World
```s
.global _start

.text
_start:
        # write(1, message, 14)
        mov $1, %rax    # system call number is 1 (write)
        mov $1, %rdi    # file handler is 1 (stdout)
        mov $msg, %rsi  # address of string to output
        mov $len, %rdx  # number of bytes
        syscall         # invoke system call

        # exit(0)
        mov $60, %rax   # system call number is 60 (exit)
        xor %rdi, %rdi  # return code is 0
        syscall         # invoke system call

.data
msg:
        .ascii "Hello, world!\n"
        len = . - msg
```
```sh
$ gcc -nostdlib hello.s && ./a.out
Hello, world!
```



## Links
- [x86 calling conventions - Wikipedia](https://en.wikipedia.org/wiki/X86_calling_conventions)
- [System V ABI - OSDev Wiki](https://wiki.osdev.org/System_V_ABI)
- [System V Application Binary Interface Intel386 Architecture Processor Supplement](https://www.uclibc.org/docs/psABI-i386.pdf)
- [System V Application Binary Interface AMD64 Architecture Processor Supplement](https://www.uclibc.org/docs/psABI-x86_64.pdf)
- [NASM Manual](https://www.csie.ntu.edu.tw/~comp03/nasm/nasmdoc0.html)
