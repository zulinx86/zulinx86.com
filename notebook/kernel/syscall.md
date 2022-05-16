---
layout: post
title: System Call
---


## How to Invoke System Calls in x86
- 32 bit user space
	- `INT 0x80` / `IRET`
		- Uses a software interrupt.
		- As this mechanism is not specialized for system call, it has many overhead.
	- `SYSENTER` / `SYSEXIT`
		- Provides a fast (low overhead) mechanism for calling system calls.
		- For `SYSENTER`,
			- `CS (target code segment selector) = IA32_SYSENTER_CS`.
			- `EIP (target instruction pointer) = IA32_SYSENTER_EIP`.
			- `SS (target stack segment selector) = IA32_SYSENTER_CS + 8`.
			- `ESP (target stack pointer) = IA32_SYSENTER_ESP`.
		- For `SYSEXIT`,
			- `CS (target code segment selector) = IA32_SYSENTER_CS + 16`.
			- `EIP (target instruction pointer) = EDX`.
			- `SS (target stack segment selector) = IA32_SYSENTER_CS + 24`.
			- `ESP (target stack pointer) = ECX`.
		- The application or the C library wrapper must set as follows before calling `SYSENTER`:
			- `ECX = ESP`.
			- `EDX = EIP`.
- 64 bit user space
	- `SYSCALL` / `SYSRET`
		- 64 bit version of `SYSENTER` / `SYSEXIT`.
		- For `SYSCALL`,
			- `R11 = RFLAGS`.
			- `RCX = RIP`.
			- `CS (target code segment selector) = IA32_STAR[47:32]`.
			- `RIP (target instruction pointer) = IA32_LSTAR`.
			- `SS (target stack segment selector) = IA32_STAR[47:32] + 8`.
			- `RFLAGS (target flag register) = RFLAGS AND NOT(IA32_FMASK)`.
		- For `SYSRET`,
			- Return to 64-bit mode (Opcode = 0F 07)
				- `CS (target code segment selector) = IA32_STAR[63:48] + 16`.
				- `RIP (target instruction pointer) = RCX`.
				- `SS (target stack segment selector) = IA32_STAR[63:48] + 8`.
				- `RFLAGS (target flag register) = R11`.
			- Return to compatibility mode (Opcode = REX.W + 0F 07)
				- `CS (target code segment selector) = IA32_STAR[63:48]`.
				- `EIP (target instruction pointer) = ECX`.
				- `SS (target stack segment selector) = IA32_STAR[63:48] + 8`.
				- `EFLAGS (target flag register) = R11`.
		- `SYSCALL` does not save the stack pointer, and `SYSRET` does not restore. Instead, the system call handler does.
- vsyscall (obsolete) vs. vDSO
	- Some system calls (e.g. `gettimeofday()`) just read a small amount of information from the kernel.
	- The transition between privilege levels becomes overhead of them.
	- The "vsyscall" and "vDSO" speed up some of these read-only syscalls by mapping the page containing the relevant information into user space as read-only.
	- vsyscall (virtual system call)
		- The vsyscall's page appears as an ELF object at a fixed position to user space, which leads to security vulnerability.
		- Some code in the vsyscall page has been removed and replaced by a special trap instruction. An application trying to call into the vsyscall page will trap into the kernel, which will then emulate the desired virtual system call in kernel space. vsyscall takes a fraction of a microsecond longer to execute, but does not break the existing ABI. In any case, the slowdown will only be seen if the application is trying to use the vsyscall page instead of the vDSO.
	- vDSO (virtual Dynamic Shared Object)
		- The vDSO's page appears as an ELF dynamic shared library at a randomized position to user space (ASLR: Address-Space Layout Randomization).
	- As you can see below, vsyscall page is always mapped to the same position, but vDSO to different positions every execution.

```
$ sudo cat /proc/self/maps
...
7ffff9b85000-7ffff9b87000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

$ sudo cat /proc/self/maps
...
7fffbcfe8000-7fffbcfea000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```



```
                            +------------------------+                                      +-------------------+
                            |    32-bit user space   |                                      | 64-bit user space |
                            |+----------++----------+|                                      |    +---------+    |
                            || EAX = 3  || EAX = 3  ||                                      |    | RAX = 0 |    |
      +----------------------+ INT 0x80 || SYSENTER +---------------------+                 |    | SYSCALL |    |
      |                     |+-------+--++----+-----+|                    |                 |    +----+----+    |
      |                     +--------|--------|------+                    |                 +---------|---------+
      |                          +---|--------+                           |                           |
      |                          |   +--------------+                     |                           |
+-----|--------------------------|------+ +---------|---------------------|---------------------------|---------+
|     |  kernel: x86_32 assembly |      | |         |              kernel:|x86_64 assembly            |         |
|+----v-----------++-------------v-----+| |+--------v-------------++------v----------------++---------v--------+|
|| entry_INT80_32 || entry_SYSENTER_32 || ||  entry_INT80_compat  || entry_SYSENTER_compat || entry_SYSCALL_64 ||
|| in entry_32.S  ||   in entry_32.S   || || in entry_64_compat.S || in entry_64_compat.S  ||  in entry_64.S   ||
|+----------------++-------------------+| |+----------------------++-----------------------++------------------+|
+---------------------------------------+ +---------------------------------------------------------------------+
```



## Definition of System Calls
### `SYSCALL_DEFINEn()` except for `SYSCALL_DEFINE0()`
[https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h#L217](https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h#L217)
```c
#define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE5(name, ...) SYSCALL_DEFINEx(5, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE6(name, ...) SYSCALL_DEFINEx(6, _##name, __VA_ARGS__)

#define SYSCALL_DEFINE_MAXARGS	6

#define SYSCALL_DEFINEx(x, sname, ...)				\
	SYSCALL_METADATA(sname, x, __VA_ARGS__)			\
	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
```
- `SYSCALL_DEFINE1()` through `SYSCALL_DEFINE6()` macros are wrappers of `SYSCALL_DEFINEx()`.
- `SYSCALL_DEFINEx()` macro calls `SYSCALL_METADATA()` and `__SYSCALL_DEFINEx()`.


### `SYSCALL_METADATA()`
[https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h#L170](https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h#L170)
```c
#ifdef CONFIG_FTRACE_SYSCALLS
/* snipped */

#define SYSCALL_METADATA(sname, nb, ...)			\
	static const char *types_##sname[] = {			\
		__MAP(nb,__SC_STR_TDECL,__VA_ARGS__)		\
	};							\
	static const char *args_##sname[] = {			\
		__MAP(nb,__SC_STR_ADECL,__VA_ARGS__)		\
	};							\
	SYSCALL_TRACE_ENTER_EVENT(sname);			\
	SYSCALL_TRACE_EXIT_EVENT(sname);			\
	static struct syscall_metadata __used			\
	  __syscall_meta_##sname = {				\
		.name 		= "sys"#sname,			\
		.syscall_nr	= -1,	/* Filled in at boot */	\
		.nb_args 	= nb,				\
		.types		= nb ? types_##sname : NULL,	\
		.args		= nb ? args_##sname : NULL,	\
		.enter_event	= &event_enter_##sname,		\
		.exit_event	= &event_exit_##sname,		\
		.enter_fields	= LIST_HEAD_INIT(__syscall_meta_##sname.enter_fields), \
	};							\
	static struct syscall_metadata __used			\
	  __section("__syscalls_metadata")			\
	 *__p_syscall_meta_##sname = &__syscall_meta_##sname;

/* snipped */
#else
#define SYSCALL_METADATA(sname, nb, ...)

/* snipped */
#endif
```
- `SYSCALL_METADATA()` is expanded only when `CONFIG_FTRACE_SYSCALLS` is defined.


### `__SYSCALL_DEFINEx()`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall_wrapper.h#L14](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall_wrapper.h#L14)
```c
/*
 * Instead of the generic __SYSCALL_DEFINEx() definition, the x86 version takes
 * struct pt_regs *regs as the only argument of the syscall stub(s) named as:
 * __x64_sys_*()         - 64-bit native syscall
 * __ia32_sys_*()        - 32-bit native syscall or common compat syscall
 * __ia32_compat_sys_*() - 32-bit compat syscall
 * __x64_compat_sys_*()  - 64-bit X32 compat syscall
 *
 * The registers are decoded according to the ABI:
 * 64-bit: RDI, RSI, RDX, R10, R8, R9
 * 32-bit: EBX, ECX, EDX, ESI, EDI, EBP
 *
 * The stub then passes the decoded arguments to the __se_sys_*() wrapper to
 * perform sign-extension (omitted for zero-argument syscalls).  Finally the
 * arguments are passed to the __do_sys_*() function which is the actual
 * syscall.  These wrappers are marked as inline so the compiler can optimize
 * the functions where appropriate.
 *
 * Example assembly (slightly re-ordered for better readability):
 *
 * <__x64_sys_recv>:		<-- syscall with 4 parameters
 *	callq	<__fentry__>
 *
 *	mov	0x70(%rdi),%rdi	<-- decode regs->di
 *	mov	0x68(%rdi),%rsi	<-- decode regs->si
 *	mov	0x60(%rdi),%rdx	<-- decode regs->dx
 *	mov	0x38(%rdi),%rcx	<-- decode regs->r10
 *
 *	xor	%r9d,%r9d	<-- clear %r9
 *	xor	%r8d,%r8d	<-- clear %r8
 *
 *	callq	__sys_recvfrom	<-- do the actual work in __sys_recvfrom()
 *				    which takes 6 arguments
 *
 *	cltq			<-- extend return value to 64-bit
 *	retq			<-- return
 *
 * This approach avoids leaking random user-provided register content down
 * the call chain.
 */
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall_wrapper.h#L228](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall_wrapper.h#L228)
```c
#define __SYSCALL_DEFINEx(x, name, ...)					\
	static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));	\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));\
	__X64_SYS_STUBx(x, name, __VA_ARGS__)				\
	__IA32_SYS_STUBx(x, name, __VA_ARGS__)				\
	static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))	\
	{								\
		long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));\
		__MAP(x,__SC_TEST,__VA_ARGS__);				\
		__PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));	\
		return ret;						\
	}								\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```
- `__SYSCALL_DEFINEx()` declares two functions:
	- `static long static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));`
	- `static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));`

[https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h#L117](https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h#L117)
```c
/*
 * __MAP - apply a macro to syscall arguments
 * __MAP(n, m, t1, a1, t2, a2, ..., tn, an) will expand to
 *    m(t1, a1), m(t2, a2), ..., m(tn, an)
 * The first argument must be equal to the amount of type/name
 * pairs given.  Note that this list of pairs (i.e. the arguments
 * of __MAP starting at the third one) is in the same format as
 * for SYSCALL_DEFINE<n>/COMPAT_SYSCALL_DEFINE<n>
 */
#define __MAP0(m,...)
#define __MAP1(m,t,a,...) m(t,a)
#define __MAP2(m,t,a,...) m(t,a), __MAP1(m,__VA_ARGS__)
#define __MAP3(m,t,a,...) m(t,a), __MAP2(m,__VA_ARGS__)
#define __MAP4(m,t,a,...) m(t,a), __MAP3(m,__VA_ARGS__)
#define __MAP5(m,t,a,...) m(t,a), __MAP4(m,__VA_ARGS__)
#define __MAP6(m,t,a,...) m(t,a), __MAP5(m,__VA_ARGS__)
#define __MAP(n,...) __MAP##n(__VA_ARGS__)
```
- As explained in the comment, `__MAP()` apply a macro to given args.
- `__MAP(n, m, t1, a1, t2, a2, ..., tn, an)` will be expanded to `m(t1,a1), m(t2,a2), ..., m(tn,an)`.

```c
#define __SC_DECL(t, a)	t a
/* snipped */
#define __TYPE_IS_LL(t) (__TYPE_AS(t, 0LL) || __TYPE_AS(t, 0ULL))
#define __SC_LONG(t, a) __typeof(__builtin_choose_expr(__TYPE_IS_LL(t), 0LL, 0L)) a
```
- `static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));` will be expanded to `static inline long __do_sys##name(t1 a1, t2 a2, ..., tx ax);`.


### `SYSCALL_DEFINE0()`
[https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h#L209](https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h#L209)
```c
#ifndef SYSCALL_DEFINE0
#define SYSCALL_DEFINE0(sname)					\
	SYSCALL_METADATA(_##sname, 0);				\
	asmlinkage long sys_##sname(void);			\
	ALLOW_ERROR_INJECTION(sys_##sname, ERRNO);		\
	asmlinkage long sys_##sname(void)
#endif /* SYSCALL_DEFINE0 */
```
- `SYSCALL_DEFINE0()` has the default implementation.
- It might be overrided depending on its architecture. x86 overrides it.

[https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h#L217](https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h#L217)
```c
#ifdef CONFIG_ARCH_HAS_SYSCALL_WRAPPER
/*
 * It may be useful for an architecture to override the definitions of the
 * SYSCALL_DEFINE0() and __SYSCALL_DEFINEx() macros, in particular to use a
 * different calling convention for syscalls. To allow for that, the prototypes
 * for the sys_*() functions below will *not* be included if
 * CONFIG_ARCH_HAS_SYSCALL_WRAPPER is enabled.
 */
#include <asm/syscall_wrapper.h>
#endif /* CONFIG_ARCH_HAS_SYSCALL_WRAPPER */
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall_wrapper.h#L249](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall_wrapper.h#L249)
```c
/*
 * As the generic SYSCALL_DEFINE0() macro does not decode any parameters for
 * obvious reasons, and passing struct pt_regs *regs to it in %rdi does not
 * hurt, we only need to re-define it here to keep the naming congruent to
 * SYSCALL_DEFINEx() -- which is essential for the COND_SYSCALL() and SYS_NI()
 * macros to work correctly.
 */
#define SYSCALL_DEFINE0(sname)						\
	SYSCALL_METADATA(_##sname, 0);					\
	static long __do_sys_##sname(const struct pt_regs *__unused);	\
	__X64_SYS_STUB0(sname)						\
	__IA32_SYS_STUB0(sname)						\
	static long __do_sys_##sname(const struct pt_regs *__unused)
```


### `__X64_SYS_STUBx()` / `__IA32_SYS_STUBx()`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall_wrapper.h#L92](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall_wrapper.h#L92)
```c
#ifdef CONFIG_X86_64
#define __X64_SYS_STUB0(name)						\
	__SYS_STUB0(x64, sys_##name)

#define __X64_SYS_STUBx(x, name, ...)					\
	__SYS_STUBx(x64, sys##name,					\
		    SC_X86_64_REGS_TO_ARGS(x, __VA_ARGS__))
/*snipped */
#else /* CONFIG_X86_64 */
#define __X64_SYS_STUB0(name)
#define __X64_SYS_STUBx(x, name, ...)
/* snipped */
#endif /* CONFIG_X86_64 */
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall_wrapper.h#L112](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall_wrapper.h#L112)
```c
#if defined(CONFIG_X86_32) || defined(CONFIG_IA32_EMULATION)
#define __IA32_SYS_STUB0(name)						\
	__SYS_STUB0(ia32, sys_##name)

#define __IA32_SYS_STUBx(x, name, ...)					\
	__SYS_STUBx(ia32, sys##name,					\
		    SC_IA32_REGS_TO_ARGS(x, __VA_ARGS__))
/* snipped */
#else /* CONFIG_X86_32 || CONFIG_IA32_EMULATION */
#define __IA32_SYS_STUB0(name)
#define __IA32_SYS_STUBx(x, name, ...)
/* snipped */
#endif /* CONFIG_X86_32 || CONFIG_IA32_EMULATION */
```

```c
#define __SYS_STUB0(abi, name)						\
	long __##abi##_##name(const struct pt_regs *regs);		\
	ALLOW_ERROR_INJECTION(__##abi##_##name, ERRNO);			\
	long __##abi##_##name(const struct pt_regs *regs)		\
		__alias(__do_##name);

#define __SYS_STUBx(abi, name, ...)					\
	long __##abi##_##name(const struct pt_regs *regs);		\
	ALLOW_ERROR_INJECTION(__##abi##_##name, ERRNO);			\
	long __##abi##_##name(const struct pt_regs *regs)		\
	{								\
		return __se_##name(__VA_ARGS__);			\
	}
```

- As it turns out, `__<abi>_sys_<name>()` -> `__se_sys_<name>()` -> `__do_sys_<name>()`.



## System Call Number
### System Call Number Table
[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/syscalls/syscall_32.tbl](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/syscalls/syscall_32.tbl)
```
#
# 32-bit system call numbers and entry vectors
#
# The format is:
# <number> <abi> <name> <entry point> <compat entry point>
#
# The __ia32_sys and __ia32_compat_sys stubs are created on-the-fly for
# sys_*() system calls and compat_sys_*() compat system calls if
# IA32_EMULATION is defined, and expect struct pt_regs *regs as their only
# parameter.
#
# The abi is always "i386" for this file.
#
0	i386	restart_syscall		sys_restart_syscall
1	i386	exit			sys_exit
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/syscalls/syscall_64.tbl](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/syscalls/syscall_64.tbl)
```
#
# 64-bit system call numbers and entry vectors
#
# The format is:
# <number> <abi> <name> <entry point>
#
# The __x64_sys_*() stubs are created on-the-fly for sys_*() system calls
#
# The abi is "common", "64" or "x32" for this file.
#
0	common	read			sys_read
1	common	write			sys_write
```

- These `.tbl` files will be compiled to `arch/x86/include/generated/asm/syscalls_32.h` and `arch/x86/include/generated/asm/syscalls_64.h` as follows:
	- arch/x86/include/generated/asm/syscalls_32.h
	```
	$ head arch/x86/include/generated/asm/syscalls_32.h
	__SYSCALL(0, sys_restart_syscall)
	__SYSCALL(1, sys_exit)
	```
	- arch/x86/include/generated/asm/syscalls_64.h
	```
	$ head arch/x86/include/generated/asm/syscalls_64.h
	__SYSCALL(0, sys_read)
	__SYSCALL(1, sys_write)
	```


### `__SYSCALL()`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/syscall_32.c#L16](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/syscall_32.c#L16)
```c
#define __SYSCALL(nr, sym) extern long __ia32_##sym(const struct pt_regs *);

#include <asm/syscalls_32.h>
#undef __SYSCALL

#define __SYSCALL(nr, sym) __ia32_##sym,

__visible const sys_call_ptr_t ia32_sys_call_table[] = {
#include <asm/syscalls_32.h>
};
```


### `sys_call_table[]`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/syscall_64.c#L10](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/syscall_64.c#L10)
```c
#define __SYSCALL(nr, sym) extern long __x64_##sym(const struct pt_regs *);
#include <asm/syscalls_64.h>
#undef __SYSCALL

#define __SYSCALL(nr, sym) __x64_##sym,

asmlinkage const sys_call_ptr_t sys_call_table[] = {
#include <asm/syscalls_64.h>
};
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall.h#L19](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall.h#L19)
```c
typedef long (*sys_call_ptr_t)(const struct pt_regs *);
extern const sys_call_ptr_t sys_call_table[];
```



## Execution Path for `SYSCALL` in x86_64 kernel
- As it turns out, the execution path of `read` syscall is as follows:
	- `entry_SYSCALL_64`
	- `do_syscall_64()`
	- `do_syscall_x64()`
	- `__x86_sys_read()`
	- `__se_sys_read()`
	- `__do_sys_read()`
	- `ksys_read()`

```
(gdb) bt
#0  ksys_read (fd=3, buf=0x7ffe331814a8 "", count=832) at fs/read_write.c:609
#1  0xffffffff81bdb33a in do_syscall_x64 (nr=<optimized out>, regs=0xffffc90000013f58)
    at arch/x86/entry/common.c:50
#2  do_syscall_64 (regs=0xffffc90000013f58, nr=<optimized out>) at arch/x86/entry/common.c:80
#3  0xffffffff81c0007c in entry_SYSCALL_64 () at arch/x86/entry/entry_64.S:113
```

### `syscall_init()`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/cpu/common.c#L1804](https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/cpu/common.c#L1804)
```c
/* May not be marked __init: used by software suspend */
void syscall_init(void)
{
	wrmsr(MSR_STAR, 0, (__USER32_CS << 16) | __KERNEL_CS);
	wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);

#ifdef CONFIG_IA32_EMULATION
	wrmsrl_cstar((unsigned long)entry_SYSCALL_compat);
	/*
	 * This only works on Intel CPUs.
	 * On AMD CPUs these MSRs are 32-bit, CPU truncates MSR_IA32_SYSENTER_EIP.
	 * This does not cause SYSENTER to jump to the wrong location, because
	 * AMD doesn't allow SYSENTER in long mode (either 32- or 64-bit).
	 */
	wrmsrl_safe(MSR_IA32_SYSENTER_CS, (u64)__KERNEL_CS);
	wrmsrl_safe(MSR_IA32_SYSENTER_ESP,
		    (unsigned long)(cpu_entry_stack(smp_processor_id()) + 1));
	wrmsrl_safe(MSR_IA32_SYSENTER_EIP, (u64)entry_SYSENTER_compat);
#else
	wrmsrl_cstar((unsigned long)ignore_sysret);
	wrmsrl_safe(MSR_IA32_SYSENTER_CS, (u64)GDT_ENTRY_INVALID_SEG);
	wrmsrl_safe(MSR_IA32_SYSENTER_ESP, 0ULL);
	wrmsrl_safe(MSR_IA32_SYSENTER_EIP, 0ULL);
#endif

	/*
	 * Flags to clear on syscall; clear as much as possible
	 * to minimize user space-kernel interference.
	 */
	wrmsrl(MSR_SYSCALL_MASK,
	       X86_EFLAGS_CF|X86_EFLAGS_PF|X86_EFLAGS_AF|
	       X86_EFLAGS_ZF|X86_EFLAGS_SF|X86_EFLAGS_TF|
	       X86_EFLAGS_IF|X86_EFLAGS_DF|X86_EFLAGS_OF|
	       X86_EFLAGS_IOPL|X86_EFLAGS_NT|X86_EFLAGS_RF|
	       X86_EFLAGS_AC|X86_EFLAGS_ID);
}
```
- When it boots up, `entry_SYSCALL_64` is set as the entry point of system calls.


### `entry_SYSCALL_64`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/entry_64.S#L50](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/entry_64.S#L50)
```c
/*
 * 64-bit SYSCALL instruction entry. Up to 6 arguments in registers.
 *
 * This is the only entry point used for 64-bit system calls.  The
 * hardware interface is reasonably well designed and the register to
 * argument mapping Linux uses fits well with the registers that are
 * available when SYSCALL is used.
 *
 * SYSCALL instructions can be found inlined in libc implementations as
 * well as some other programs and libraries.  There are also a handful
 * of SYSCALL instructions in the vDSO used, for example, as a
 * clock_gettimeofday fallback.
 *
 * 64-bit SYSCALL saves rip to rcx, clears rflags.RF, then saves rflags to r11,
 * then loads new ss, cs, and rip from previously programmed MSRs.
 * rflags gets masked by a value from another MSR (so CLD and CLAC
 * are not needed). SYSCALL does not save anything on the stack
 * and does not change rsp.
 *
 * Registers on entry:
 * rax  system call number
 * rcx  return address
 * r11  saved rflags (note: r11 is callee-clobbered register in C ABI)
 * rdi  arg0
 * rsi  arg1
 * rdx  arg2
 * r10  arg3 (needs to be moved to rcx to conform to C ABI)
 * r8   arg4
 * r9   arg5
 * (note: r12-r15, rbp, rbx are callee-preserved in C ABI)
 *
 * Only called from user space.
 *
 * When user can change pt_regs->foo always force IRET. That is because
 * it deals with uncanonical addresses better. SYSRET has trouble
 * with them due to bugs in both AMD and Intel CPUs.
 */
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/entry_64.S#L87](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/entry_64.S#L87)
```
SYM_CODE_START(entry_SYSCALL_64)
/* snipped */
	/* IRQs are off. */
	movq	%rsp, %rdi
	/* Sign extend the lower 32bit as syscall numbers are treated as int */
	movslq	%eax, %rsi
	call	do_syscall_64		/* returns with IRQs disabled */
/* snipped */
	sysretq
SYM_CODE_END(entry_SYSCALL_64)
```


### `do_syscall_64()`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall.h#L129](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/syscall.h#L129)
```c
void do_syscall_64(struct pt_regs *regs, int nr);
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L73](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L73)
```c
__visible noinstr void do_syscall_64(struct pt_regs *regs, int nr)
{
	add_random_kstack_offset();
	nr = syscall_enter_from_user_mode(regs, nr);

	instrumentation_begin();

	if (!do_syscall_x64(regs, nr) && !do_syscall_x32(regs, nr) && nr != -1) {
		/* Invalid system call, but still a system call. */
		regs->ax = __x64_sys_ni_syscall(regs);
	}

	instrumentation_end();
	syscall_exit_to_user_mode(regs);
}
```


### `struct pt_regs`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/ptrace.h#L59](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/ptrace.h#L59)
```c
struct pt_regs {
/*
 * C ABI says these regs are callee-preserved. They aren't saved on kernel entry
 * unless syscall needs a complete, fully filled "struct pt_regs".
 */
	unsigned long r15;
	unsigned long r14;
	unsigned long r13;
	unsigned long r12;
	unsigned long bp;
	unsigned long bx;
/* These regs are callee-clobbered. Always saved on kernel entry. */
	unsigned long r11;
	unsigned long r10;
	unsigned long r9;
	unsigned long r8;
	unsigned long ax;
	unsigned long cx;
	unsigned long dx;
	unsigned long si;
	unsigned long di;
/*
 * On syscall entry, this is syscall#. On CPU exception, this is error code.
 * On hw interrupt, it's IRQ number:
 */
	unsigned long orig_ax;
/* Return frame for iretq */
	unsigned long ip;
	unsigned long cs;
	unsigned long flags;
	unsigned long sp;
	unsigned long ss;
/* top of stack page */
};
```


### `do_syscall_x64()`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L40](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L40)
```c
static __always_inline bool do_syscall_x64(struct pt_regs *regs, int nr)
{
	/*
	 * Convert negative numbers to very high and thus out of range
	 * numbers for comparisons.
	 */
	unsigned int unr = nr;

	if (likely(unr < NR_syscalls)) {
		unr = array_index_nospec(unr, NR_syscalls);
		regs->ax = sys_call_table[unr](regs);
		return true;
	}
	return false;
}
```

- For `read` syscall, `sys_call_table[unr]` is `sys_call_table[0]` pointing to `__x86_sys_read(regs)`.


### `SYSCALL_DEFINE3(read, ...)`
[https://elixir.bootlin.com/linux/latest/source/fs/read_write.c#L627](https://elixir.bootlin.com/linux/latest/source/fs/read_write.c#L627)
```c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
	return ksys_read(fd, buf, count);
}
```


### `ksys_read()`
[https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h#L1293](https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h#L1293)
```c
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count);
```

[https://elixir.bootlin.com/linux/latest/source/fs/read_write.c#L608](https://elixir.bootlin.com/linux/latest/source/fs/read_write.c#L608)
```c
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos, *ppos = file_ppos(f.file);
		if (ppos) {
			pos = *ppos;
			ppos = &pos;
		}
		ret = vfs_read(f.file, buf, count, ppos);
		if (ret >= 0 && ppos)
			f.file->f_pos = pos;
		fdput_pos(f);
	}
	return ret;
}
```



## Execution Path for `SYSENTER` in x86_64 kernel
### `entry_SYSENTER_compat()`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/proto.h#L27](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/proto.h#L27)
```c
void entry_SYSENTER_compat(void);
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/entry_64_compat.S#L49](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/entry_64_compat.S#L49)
```
/*
 * 32-bit SYSENTER entry.
 *
 * 32-bit system calls through the vDSO's __kernel_vsyscall enter here
 * on 64-bit kernels running on Intel CPUs.
 *
 * The SYSENTER instruction, in principle, should *only* occur in the
 * vDSO.  In practice, a small number of Android devices were shipped
 * with a copy of Bionic that inlined a SYSENTER instruction.  This
 * never happened in any of Google's Bionic versions -- it only happened
 * in a narrow range of Intel-provided versions.
 *
 * SYSENTER loads SS, RSP, CS, and RIP from previously programmed MSRs.
 * IF and VM in RFLAGS are cleared (IOW: interrupts are off).
 * SYSENTER does not save anything on the stack,
 * and does not save old RIP (!!!), RSP, or RFLAGS.
 *
 * Arguments:
 * eax  system call number
 * ebx  arg1
 * ecx  arg2
 * edx  arg3
 * esi  arg4
 * edi  arg5
 * ebp  user stack
 * 0(%ebp) arg6
 */
SYM_CODE_START(entry_SYSENTER_compat)
/* snipped */
	movq	%rsp, %rdi
	call	do_SYSENTER_32
/* snipped */
	jmp	sysret32_from_system_call
/* snipped */
SYM_CODE_END(entry_SYSENTER_compat)
```


### `do_SYSENTER_32()`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L238](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L238)
```c
/* Returns 0 to return using IRET or 1 to return using SYSEXIT/SYSRETL. */
__visible noinstr long do_SYSENTER_32(struct pt_regs *regs)
{
	/* SYSENTER loses RSP, but the vDSO saved it in RBP. */
	regs->sp = regs->bp;

	/* SYSENTER clobbers EFLAGS.IF.  Assume it was set in usermode. */
	regs->flags |= X86_EFLAGS_IF;

	return do_fast_syscall_32(regs);
}
```


### `do_fast_syscall_32()`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L186](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L186)
```c
/* Returns 0 to return using IRET or 1 to return using SYSEXIT/SYSRETL. */
__visible noinstr long do_fast_syscall_32(struct pt_regs *regs)
{
/* snipped */
	/* Invoke the syscall. If it failed, keep it simple: use IRET. */
	if (!__do_fast_syscall_32(regs))
		return 0;
/* snipped */
}
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L138](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L138)
```c
static noinstr bool __do_fast_syscall_32(struct pt_regs *regs)
{
/* snipped */
	/* Now this is just like a normal syscall. */
	do_syscall_32_irqs_on(regs, nr);
/* snipped */
	return true;
}
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L102](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L102)
```c
/*
 * Invoke a 32-bit syscall.  Called with IRQs on in CONTEXT_KERNEL.
 */
static __always_inline void do_syscall_32_irqs_on(struct pt_regs *regs, int nr)
{
	/*
	 * Convert negative numbers to very high and thus out of range
	 * numbers for comparisons.
	 */
	unsigned int unr = nr;

	if (likely(unr < IA32_NR_syscalls)) {
		unr = array_index_nospec(unr, IA32_NR_syscalls);
		regs->ax = ia32_sys_call_table[unr](regs);
	} else if (nr != -1) {
		regs->ax = __ia32_sys_ni_syscall(regs);
	}
}
```



## Execution Path for `INT 0x80` in x86_64 kernel
### `def_idts[]`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/idt.c#L112](https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/idt.c#L112)
```c
/*
 * The default IDT entries which are set up in trap_init() before
 * cpu_init() is invoked. Interrupt stacks cannot be used at that point and
 * the traps which use them are reinitialized with IST after cpu_init() has
 * set up TSS.
 */
static const __initconst struct idt_data def_idts[] = {
/* snipped */
#if defined(CONFIG_IA32_EMULATION)
	SYSG(IA32_SYSCALL_VECTOR,	entry_INT80_compat),
#elif defined(CONFIG_X86_32)
	SYSG(IA32_SYSCALL_VECTOR,	entry_INT80_32),
#endif
};
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/irq_vectors.h#L45](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/irq_vectors.h#L45)
```c
#define IA32_SYSCALL_VECTOR		0x80
```


### `entry_INT80_compat()`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/entry_64_compat.S#L341](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/entry_64_compat.S#L341)
```
/*
 * 32-bit legacy system call entry.
 *
 * 32-bit x86 Linux system calls traditionally used the INT $0x80
 * instruction.  INT $0x80 lands here.
 *
 * This entry point can be used by 32-bit and 64-bit programs to perform
 * 32-bit system calls.  Instances of INT $0x80 can be found inline in
 * various programs and libraries.  It is also used by the vDSO's
 * __kernel_vsyscall fallback for hardware that doesn't support a faster
 * entry method.  Restarted 32-bit system calls also fall back to INT
 * $0x80 regardless of what instruction was originally used to do the
 * system call.
 *
 * This is considered a slow path.  It is not used by most libc
 * implementations on modern hardware except during process startup.
 *
 * Arguments:
 * eax  system call number
 * ebx  arg1
 * ecx  arg2
 * edx  arg3
 * esi  arg4
 * edi  arg5
 * ebp  arg6
 */
SYM_CODE_START(entry_INT80_compat)
/* snipped */
	movq	%rsp, %rdi
	call	do_int80_syscall_32
/* snipped */
SYM_CODE_END(entry_INT80_compat)
```


### `do_int80_syscall_32()`
[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L119](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L119)
```c
/* Handles int $0x80 */
__visible noinstr void do_int80_syscall_32(struct pt_regs *regs)
{
	int nr = syscall_32_enter(regs);
/* snipped */
	do_syscall_32_irqs_on(regs, nr);
/* snipped */
}
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L102](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L102)
```c
/*
 * Invoke a 32-bit syscall.  Called with IRQs on in CONTEXT_KERNEL.
 */
static __always_inline void do_syscall_32_irqs_on(struct pt_regs *regs, int nr)
{
	/*
	 * Convert negative numbers to very high and thus out of range
	 * numbers for comparisons.
	 */
	unsigned int unr = nr;

	if (likely(unr < IA32_NR_syscalls)) {
		unr = array_index_nospec(unr, IA32_NR_syscalls);
		regs->ax = ia32_sys_call_table[unr](regs);
	} else if (nr != -1) {
		regs->ax = __ia32_sys_ni_syscall(regs);
	}
}
```



## vDSO
### Linker Scripts
[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/vdso/vdso.lds.S](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/vdso/vdso.lds.S)
```c
/*
 * This controls what userland symbols we export from the vDSO.
 */
VERSION {
	LINUX_2.6 {
	global:
		clock_gettime;
		__vdso_clock_gettime;
		gettimeofday;
		__vdso_gettimeofday;
		getcpu;
		__vdso_getcpu;
		time;
		__vdso_time;
		clock_getres;
		__vdso_clock_getres;
		__vdso_sgx_enter_enclave;
	local: *;
	};
}
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/vdso/vdso-layout.lds.S](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/vdso/vdso-layout.lds.S)


### Make the vDSO Page Accessible
[https://elixir.bootlin.com/linux/latest/source/fs/binfmt_elf.c#L100](https://elixir.bootlin.com/linux/latest/source/fs/binfmt_elf.c#L100)
```c
static struct linux_binfmt elf_format = {
	.module		= THIS_MODULE,
	.load_binary	= load_elf_binary,
	.load_shlib	= load_elf_library,
	.core_dump	= elf_core_dump,
	.min_coredump	= ELF_EXEC_PAGESIZE,
};
```

[https://elixir.bootlin.com/linux/latest/source/fs/binfmt_elf.c#L823](https://elixir.bootlin.com/linux/latest/source/fs/binfmt_elf.c#L823)
```c
static int load_elf_binary(struct linux_binprm *bprm)
{
/* snipped */
#ifdef ARCH_HAS_SETUP_ADDITIONAL_PAGES
	retval = ARCH_SETUP_ADDITIONAL_PAGES(bprm, elf_ex, !!interpreter);
	if (retval < 0)
		goto out;
#endif /* ARCH_HAS_SETUP_ADDITIONAL_PAGES */
/* snipped */
}
```

[https://elixir.bootlin.com/linux/latest/source/include/linux/elf.h#L31](https://elixir.bootlin.com/linux/latest/source/include/linux/elf.h#L31)
```c
#if defined(ARCH_HAS_SETUP_ADDITIONAL_PAGES) && !defined(ARCH_SETUP_ADDITIONAL_PAGES)
#define ARCH_SETUP_ADDITIONAL_PAGES(bprm, ex, interpreter) \
	arch_setup_additional_pages(bprm, interpreter)
#endif
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/vdso/vma.c#L389](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/vdso/vma.c#L389)
```c
int arch_setup_additional_pages(struct linux_binprm *bprm, int uses_interp)
{
	if (!vdso64_enabled)
		return 0;

	return map_vdso_randomized(&vdso_image_64);
}
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/vdso/vma.c#L345](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/vdso/vma.c#L345)
```c
static int map_vdso_randomized(const struct vdso_image *image)
{
	unsigned long addr = vdso_addr(current->mm->start_stack, image->size-image->sym_vvar_start);

	return map_vdso(image, addr);
}
```

[https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/vdso/vma.c#L246](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/vdso/vma.c#L246)
```c
/*
 * Add vdso and vvar mappings to current process.
 * @image          - blob to map
 * @addr           - request a specific address (zero to map at free addr)
 */
static int map_vdso(const struct vdso_image *image, unsigned long addr)
{
/* snipped */
	if (IS_ERR(vma)) {
		ret = PTR_ERR(vma);
		do_munmap(mm, text_start, image->size, NULL);
	} else {
		current->mm->context.vdso = (void __user *)text_start;
		current->mm->context.vdso_image = image;
	}
/* snipped */
}
```



## Hands-On
### Display "Hello World" into the Standard Output
```hello.S
.data

msg:
	.ascii "Hello, world!\n"
	len = . - msg

.text
	.global _start

_start:
	movq $1, %rax		# syscall number (1 means write)
	movq $1, %rdi		# first argument (fd = 1)
	movq $msg, %rsi		# second argument (buf = msg)
	movq $len, %rdx		# third argument (count = len)
	syscall				# invoke system call

	movq $60, %rax		# syscall number (60 means exit)
	xorq %rdi, %rdi		# first argument (status = 0)
	syscall				# invoke system call
```
```
$ gcc -c hello.S
$ ld -o hello hello.o
$ ./hello
Hello, world!
```
```
$ strace ./hello
execve("./hello", ["./hello"], 0x7fffc1475080 /* 22 vars */) = 0
write(1, "Hello, world!\n", 14Hello, world!
)         = 14
exit(0)                                 = ?
+++ exited with 0 +++
```



## Links
- [Anatomy of a system call, part 1 - LWN.net](https://lwn.net/Articles/604287/)
- [Anatomy of a system call, part 2 - LWN.net](https://lwn.net/Articles/604515/)
- [On vsyscalls and the vDSO - LWN.net](https://lwn.net/Articles/446528/)
- [System calls · Linux Inside](https://0xax.gitbooks.io/linux-insides/content/SysCall/)
- [SYSENTER - OSDev Wiki](https://wiki.osdev.org/SYSENTER#Compatibility_across_Intel_and_AMD)
- [System Calls — The Linux Kernel documentation](https://linux-kernel-labs.github.io/refs/heads/master/lectures/syscalls.html)
