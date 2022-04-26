---
layout: post
title: A20 Line
---


## What is A20 Line?
- The A20 address line is the 21st bit of physical memory address.
- The 8086's address space is 1 MiB because there are 20 address lines (A0 through A19). Any address above the 1 MiB wraps around to zero.
- The latter model's address space is greater than 1 MiB, but to remain compatible with the 8086, the A20 line on the address bus was disabled by default.


## Test the A20 Line
```
	; Check if A20 line is enabled
	; Returns:
	; - 0: in ax if the a20 line is disabled (memory wraps around).
	; - 1: in ax if the a20 line is enabled (memory does not wrap around).
check_a20:
	pushf
	push ds
	push es
	push di
	push si

	xor ax,ax
	mov es,ax		; es = 0x0000
	not ax
	mov ds,ax		; ds = 0xffff

	mov di,0x0500	; di = 0x0500
	mov si,0x0510	; si = 0x0510

	mov al,[es:di]	; al = [es:di] (0x000500)
	push ax
	mov al,[ds:si]	; al = [ds:si] (0x100500)
	push ax

	mov byte [es:di],0x00
	mov byte [ds:si],0xff

	cmp byte [es:di],0xff

	pop ax
	mov [ds:si],al

	pop ax
	mov [es:di],al

	mov ax,0
	je .exit

	mov ax,1

.exit:
	pop si
	pop di
	pop es
	pop ds
	popf
	
	ret
```


## Enable the A20 Line by Keyboard Controller
```
	kbc_data equ 0x60
	kbc_comm equ 0x64
	kbc_stat equ 0x64

	; Enable A20 line
enable_a20:
	call check_a20
	cmp al,0
	jz .exit

	call wait_kbc_out
	mov al,0xd0			; function to read controller output port
	out kbc_comm,al		; send command

	call wait_kbc_in
	in al,kbc_data		; read data
	push eax

	call wait_kbc_out
	mov al,0xd1			; function to write controller out port
	out kbc_comm,al		; send command

	call wait_kbc_out
	pop eax
	or al,2				; enable a20 bit
	out kbc_data,al

	call wait_kbc_out
.exit:
	ret


wait_kbc_in:
	in al,kbc_stat
	test al,1
	jz wait_kbc_in
	ret


wait_kbc_out:
	in al,kbc_stat
	test al,2
	jnz wait_kbc_out
	ret
```


## Links
- [A20 Line - OSDev Wiki](https://wiki.osdev.org/A20_Line)