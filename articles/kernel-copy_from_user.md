---
title: 【Kernel】copy_to_user() / copy_from_user() / get_user() / put_user()
emoji: "🐧"
type: "tech"
topics: ["linux", "kernel"]
published: true
---

※ Linux kernel v6.10.1 のコードベースを元に調査したものです。



# copy_to_user() / copy_from_user() / get_user() / put_user()

- `put_user()`: ユーザー空間へ単一の値 (`int`, `char`, `long` など) を渡す
- `get_user()`: ユーザー空間から単一の値を受け取る
- `copy_to_user()`: ユーザー空間に任意の量のデータを渡す
- `copy_from_user()`: ユーザー空間から任意の量のデータを受け取る

https://www.kernel.org/doc/html/latest/kernel-hacking/hacking.html#copy-to-user-copy-from-user-get-user-put-user

> `copy_to_user()` / `copy_from_user()` / `get_user()` / `put_user()`
>
> Defined in `include/linux/uaccess.h` / `asm/uaccess.h`
> 
> [SLEEPS]
> 
> `put_user()` and `get_user()` are used to get and put single values (such as an int, char, or long) from and to userspace. A pointer into userspace should never be simply dereferenced: data should be copied using these routines. Both return -EFAULT or 0.
> 
> copy_to_user() and copy_from_user() are more general: they copy an arbitrary amount of data to and from userspace.
>
> Warning:
> Unlike put_user() and get_user(), they return the amount of uncopied data (ie. 0 still means success).
>
> [Yes, this objectionable interface makes me cringe. The flamewar comes up every year or so. --RR.]
>
> The functions may sleep implicitly. This should never be called outside user context (it makes no sense), with interrupts disabled, or a spinlock held.



# x86


## [`put_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/include/asm/uaccess.h#L191)

前述の通り、ユーザー空間に単一の値を書き込む関数である。

また、コメントにある通り、ページフォルトが有効な場合は、関数がスリープする場合がある。

```c
/**
 * put_user - Write a simple value into user space.
 * @x:   Value to copy to user space.
 * @ptr: Destination address, in user space.
 *
 * Context: User context only. This function may sleep if pagefaults are
 *          enabled.
 *
 * This macro copies a single simple value from kernel space to user
 * space.  It supports simple types like char and int, but not larger
 * data types like structures or arrays.
 *
 * @ptr must have pointer-to-simple-variable type, and @x must be assignable
 * to the result of dereferencing @ptr.
 *
 * Return: zero on success, or -EFAULT on error.
 */
#define put_user(x, ptr) ({ might_fault(); do_put_user_call(put_user,x,ptr); })
```

`might_fault()` は、やや複雑そうだったので、本記事では割愛します。後日別の記事にする可能性はありです。

### [`do_put_user_call()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/include/asm/uaccess.h#L170)

結論だけをいうと、サイズに応じて `__put_user_1()`, `__put_user_2()`, `__put_user_4()`, `__put_user_8()` のいずれかが呼び出される。

もう少しだけ具体的にアセンブリの呼び出し部分を見る。
- 返り値 `__ret_pu` は RCX レジスタに出力される。
- ユーザー空間のポインタ `__ptr_pu` は、`__ret_pu` と同様に RCX レジスタに保持する。
- ユーザー空間に渡す値 `__val_pu` は、RAX レジスタに保持する。

```c
/*
 * ptr must be evaluated and assigned to the temporary __ptr_pu before
 * the assignment of x to __val_pu, to avoid any function calls
 * involved in the ptr expression (possibly implicitly generated due
 * to KASAN) from clobbering %ax.
 */
#define do_put_user_call(fn,x,ptr)					\
({									\
	int __ret_pu;							\
	void __user *__ptr_pu;						\
	register __typeof__(*(ptr)) __val_pu asm("%"_ASM_AX);		\
	__typeof__(*(ptr)) __x = (x); /* eval x once */			\
	__typeof__(ptr) __ptr = (ptr); /* eval ptr once */		\
	__chk_user_ptr(__ptr);						\
	__ptr_pu = __ptr;						\
	__val_pu = __x;							\
	asm volatile("call __" #fn "_%c[size]"				\
		     : "=c" (__ret_pu),					\
			ASM_CALL_CONSTRAINT				\
		     : "0" (__ptr_pu),					\
		       "r" (__val_pu),					\
		       [size] "i" (sizeof(*(ptr)))			\
		     :"ebx");						\
	instrument_put_user(__x, __ptr, sizeof(*(ptr)));		\
	__builtin_expect(__ret_pu, 0);					\
})
```

### [`__put_user_1`, `__put_user_2`, `__put_user_4`, `__put_user_8`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/lib/putuser.S#L47)

`check_range` については後述。

[STAC 命令](https://www.felixcloutier.com/x86/stac) は EFLAGS レジスタの AC フラグをセットする。これはカーネルが、ユーザー空間のメモリにアクセスすることを許可するための命令である。[CLAC 命令](https://www.felixcloutier.com/x86/clac) は、逆に EFLAGS レジスタの AC フラグをクリアします。

前述の通り、RCX レジスタにはユーザー空間のポインタ、RAX レジスタにはユーザー空間に渡す値がセットされている。
`movb %al,(%_ASM_CX)` 等の場所で実際に値がコピーされている。

その後 `xor %ecx,%ecx` を実行し、ECX レジスタを 0 にセットし、返り値として `__ret_pu` にセットされる。

```c
SYM_FUNC_START(__put_user_1)
	check_range size=1
	ASM_STAC
1:	movb %al,(%_ASM_CX)
	xor %ecx,%ecx
	ASM_CLAC
	RET
SYM_FUNC_END(__put_user_1)
EXPORT_SYMBOL(__put_user_1)
```
```c
SYM_FUNC_START(__put_user_2)
	check_range size=2
	ASM_STAC
3:	movw %ax,(%_ASM_CX)
	xor %ecx,%ecx
	ASM_CLAC
	RET
SYM_FUNC_END(__put_user_2)
EXPORT_SYMBOL(__put_user_2)
```
```c
SYM_FUNC_START(__put_user_4)
	check_range size=4
	ASM_STAC
5:	movl %eax,(%_ASM_CX)
	xor %ecx,%ecx
	ASM_CLAC
	RET
SYM_FUNC_END(__put_user_4)
EXPORT_SYMBOL(__put_user_4)
```
```c
SYM_FUNC_START(__put_user_8)
	check_range size=8
	ASM_STAC
7:	mov %_ASM_AX,(%_ASM_CX)
#ifdef CONFIG_X86_32
8:	movl %edx,4(%_ASM_CX)
#endif
	xor %ecx,%ecx
	ASM_CLAC
	RET
SYM_FUNC_END(__put_user_8)
EXPORT_SYMBOL(__put_user_8)
```


### [`check_range`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/lib/putuser.S#L35)

前述の通り、RCX にはユーザースペースポインタが設定されていることが前提となる。
RBX に一旦移動し、最上位ビットを使った 64-bit のマスクを作成し、OR をとっている。
したがって、最上位ビットが 0 の場合、与えられたポインタそのままが返され、そうでない場合は `~0` のポインタが返される。

```c
.macro check_range size:req
.if IS_ENABLED(CONFIG_X86_64)
	mov %rcx, %rbx
	sar $63, %rbx
	or %rbx, %rcx
.else
	cmp $TASK_SIZE_MAX-\size+1, %ecx
	jae .Lbad_put_user
.endif
.endm
```

以下のメモリマップを考慮すると、与えられたポインタがユーザースペースのものであれば、そのままの値を返し、カーネルスペースであれば unused hole を示す `~0` を返すことになる。

https://www.kernel.org/doc/Documentation/x86/x86_64/mm.txt

```
========================================================================================================================
    Start addr    |   Offset   |     End addr     |  Size   | VM area description
========================================================================================================================
                  |            |                  |         |
 0000000000000000 |    0       | 00007fffffffffff |  128 TB | user-space virtual memory, different per mm
__________________|____________|__________________|_________|___________________________________________________________
                  |            |                  |         |
 0000800000000000 | +128    TB | ffff7fffffffffff | ~16M TB | ... huge, almost 64 bits wide hole of non-canonical
                  |            |                  |         |     virtual memory addresses up to the -128 TB
                  |            |                  |         |     starting offset of kernel mappings.
__________________|____________|__________________|_________|___________________________________________________________
                                                            |
                                                            | Kernel-space virtual memory, shared between all processes:
____________________________________________________________|___________________________________________________________
                  |            |                  |         |
 ffff800000000000 | -128    TB | ffff87ffffffffff |    8 TB | ... guard hole, also reserved for hypervisor
 ffff880000000000 | -120    TB | ffff887fffffffff |  0.5 TB | LDT remap for PTI
 ffff888000000000 | -119.5  TB | ffffc87fffffffff |   64 TB | direct mapping of all physical memory (page_offset_base)
 ffffc88000000000 |  -55.5  TB | ffffc8ffffffffff |  0.5 TB | ... unused hole
 ffffc90000000000 |  -55    TB | ffffe8ffffffffff |   32 TB | vmalloc/ioremap space (vmalloc_base)
 ffffe90000000000 |  -23    TB | ffffe9ffffffffff |    1 TB | ... unused hole
 ffffea0000000000 |  -22    TB | ffffeaffffffffff |    1 TB | virtual memory map (vmemmap_base)
 ffffeb0000000000 |  -21    TB | ffffebffffffffff |    1 TB | ... unused hole
 ffffec0000000000 |  -20    TB | fffffbffffffffff |   16 TB | KASAN shadow memory
__________________|____________|__________________|_________|____________________________________________________________
                                                            |
                                                            | Identical layout to the 56-bit one from here on:
____________________________________________________________|____________________________________________________________
                  |            |                  |         |
 fffffc0000000000 |   -4    TB | fffffdffffffffff |    2 TB | ... unused hole
                  |            |                  |         | vaddr_end for KASLR
 fffffe0000000000 |   -2    TB | fffffe7fffffffff |  0.5 TB | cpu_entry_area mapping
 fffffe8000000000 |   -1.5  TB | fffffeffffffffff |  0.5 TB | ... unused hole
 ffffff0000000000 |   -1    TB | ffffff7fffffffff |  0.5 TB | %esp fixup stacks
 ffffff8000000000 | -512    GB | ffffffeeffffffff |  444 GB | ... unused hole
 ffffffef00000000 |  -68    GB | fffffffeffffffff |   64 GB | EFI region mapping space
 ffffffff00000000 |   -4    GB | ffffffff7fffffff |    2 GB | ... unused hole
 ffffffff80000000 |   -2    GB | ffffffff9fffffff |  512 MB | kernel text mapping, mapped to physical address 0
 ffffffff80000000 |-2048    MB |                  |         |
 ffffffffa0000000 |-1536    MB | fffffffffeffffff | 1520 MB | module mapping space
 ffffffffff000000 |  -16    MB |                  |         |
    FIXADDR_START | ~-11    MB | ffffffffff5fffff | ~0.5 MB | kernel-internal fixmap range, variable size and offset
 ffffffffff600000 |  -10    MB | ffffffffff600fff |    4 kB | legacy vsyscall ABI
 ffffffffffe00000 |   -2    MB | ffffffffffffffff |    2 MB | ... unused hole
__________________|____________|__________________|_________|___________________________________________________________
```

[このパッチ](https://lore.kernel.org/all/20230312112612.31869-2-kirill.shutemov@linux.intel.com/)も併せて読むと良い。


## [`get_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/include/asm/uaccess.h#L108)

```c
/**
 * get_user - Get a simple variable from user space.
 * @x:   Variable to store result.
 * @ptr: Source address, in user space.
 *
 * Context: User context only. This function may sleep if pagefaults are
 *          enabled.
 *
 * This macro copies a single simple variable from user space to kernel
 * space.  It supports simple types like char and int, but not larger
 * data types like structures or arrays.
 *
 * @ptr must have pointer-to-simple-variable type, and the result of
 * dereferencing @ptr must be assignable to @x without a cast.
 *
 * Return: zero on success, or -EFAULT on error.
 * On error, the variable @x is set to zero.
 */
#define get_user(x,ptr) ({ might_fault(); do_get_user_call(get_user,x,ptr); })
```

### [`do_get_user_call()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/include/asm/uaccess.h#L76)

`get_user()` からも `__get_user()` からも使われる関数とのこと。
結果的にはサイズに応じた `__get_user_<size>()` を呼び出す。

アセンブリの呼び出し部分をもう少し見る。
- 返り値 `__ret_gu` は、RAX レジスタに出力される。
- ユーザーから受け取る値 `__val_gu` は、RDX レジスタに出力される。
- ユーザー空間のポインタ `ptr` は、`__ret_gu` と同様に RAX レジスタに保持する。

```c
/*
 * This is used for both get_user() and __get_user() to expand to
 * the proper special function call that has odd calling conventions
 * due to returning both a value and an error, and that depends on
 * the size of the pointer passed in.
 *
 * Careful: we have to cast the result to the type of the pointer
 * for sign reasons.
 *
 * The use of _ASM_DX as the register specifier is a bit of a
 * simplification, as gcc only cares about it as the starting point
 * and not size: for a 64-bit value it will use %ecx:%edx on 32 bits
 * (%ecx being the next register in gcc's x86 register sequence), and
 * %rdx on 64 bits.
 *
 * Clang/LLVM cares about the size of the register, but still wants
 * the base register for something that ends up being a pair.
 */
#define do_get_user_call(fn,x,ptr)					\
({									\
	int __ret_gu;							\
	register __inttype(*(ptr)) __val_gu asm("%"_ASM_DX);		\
	__chk_user_ptr(ptr);						\
	asm volatile("call __" #fn "_%c[size]"				\
		     : "=a" (__ret_gu), "=r" (__val_gu),		\
			ASM_CALL_CONSTRAINT				\
		     : "0" (ptr), [size] "i" (sizeof(*(ptr))));		\
	instrument_get_user(__val_gu);					\
	(x) = (__force __typeof__(*(ptr))) __val_gu;			\
	__builtin_expect(__ret_gu, 0);					\
})
```

### [`__get_user_1`, `__get_user_2`, `__get_user_4`, `__get_user_8`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/lib/getuser.S#L58)

`check_range` については後述。

STAC 命令と CLAC 命令は前述の通り。

前述の通り、RAX レジスタにユーザー空間のポインタ、RDX レジスタに読み込んだ値を入れる。
`movzbl (%_ASM_AX),%edx` の部分が値の読み込んでいる部分である。

最後に返り値である EAX レジスタを 0 にセットしてリターンする。

```c
	.text
SYM_FUNC_START(__get_user_1)
	check_range size=1
	ASM_STAC
1:	movzbl (%_ASM_AX),%edx
	xor %eax,%eax
	ASM_CLAC
	RET
SYM_FUNC_END(__get_user_1)
EXPORT_SYMBOL(__get_user_1)

SYM_FUNC_START(__get_user_2)
	check_range size=2
	ASM_STAC
2:	movzwl (%_ASM_AX),%edx
	xor %eax,%eax
	ASM_CLAC
	RET
SYM_FUNC_END(__get_user_2)
EXPORT_SYMBOL(__get_user_2)

SYM_FUNC_START(__get_user_4)
	check_range size=4
	ASM_STAC
3:	movl (%_ASM_AX),%edx
	xor %eax,%eax
	ASM_CLAC
	RET
SYM_FUNC_END(__get_user_4)
EXPORT_SYMBOL(__get_user_4)

SYM_FUNC_START(__get_user_8)
	check_range size=8
	ASM_STAC
#ifdef CONFIG_X86_64
4:	movq (%_ASM_AX),%rdx
#else
4:	movl (%_ASM_AX),%edx
5:	movl 4(%_ASM_AX),%ecx
#endif
	xor %eax,%eax
	ASM_CLAC
	RET
SYM_FUNC_END(__get_user_8)
EXPORT_SYMBOL(__get_user_8)
```

### [`check_range`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/lib/getuser.S#L40)

64-bit の場合は、`put_user()` と同じマスキングをしている。

32-bit の場合は、最上位ビットを使ったマスキングは使えず、`array_index_mask_nospec()` と同じ Bounds Clipping を使っている。(詳細は割愛)

```c
.macro check_range size:req
.if IS_ENABLED(CONFIG_X86_64)
	mov %rax, %rdx
	sar $63, %rdx
	or %rdx, %rax
.else
	cmp $TASK_SIZE_MAX-\size+1, %eax
.if \size != 8
	jae .Lbad_get_user
.else
	jae .Lbad_get_user_8
.endif
	sbb %edx, %edx		/* array_index_mask_nospec() */
	and %edx, %eax
.endif
.endm
```


## [`copy_to_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/include/linux/uaccess.h#L188)

```c
static __always_inline unsigned long __must_check
copy_to_user(void __user *to, const void *from, unsigned long n)
{
	if (check_copy_size(from, n, true))
		n = _copy_to_user(to, from, n);
	return n;
}
```

### [`check_copy_size()`](https://elixir.bootlin.com/linux/v6.10.1/source/include/linux/thread_info.h#L237)

一旦、この記事では深掘りしない。基本的には true が返るものと想定。

```c
static __always_inline __must_check bool
check_copy_size(const void *addr, size_t bytes, bool is_source)
{
	int sz = __builtin_object_size(addr, 0);
	if (unlikely(sz >= 0 && sz < bytes)) {
		if (!__builtin_constant_p(bytes))
			copy_overflow(sz, bytes);
		else if (is_source)
			__bad_copy_from();
		else
			__bad_copy_to();
		return false;
	}
	if (WARN_ON_ONCE(bytes > INT_MAX))
		return false;
	check_object_size(addr, bytes, is_source);
	return true;
}
```

### `_copy_to_user()`

インラインバージョンと通常バージョンが定義されている。違いとしては、`static inline __must_check` がついてるくらい。

`should_fail_usercopy()` は、ざっとみた感じは意図的にコピーを失敗させたい時に使うものっぽいのでスルー。

`access_ok()` については、[この記事](https://zenn.dev/zulinx86/articles/kernel-access_ok)を参照。

`instrument_copy_to_user()` は instrumentation 関連なので一旦スルー。

[https://elixir.bootlin.com/linux/v6.10.1/source/include/linux/uaccess.h#L163](https://elixir.bootlin.com/linux/v6.10.1/source/include/linux/uaccess.h#L163)
```c
#ifdef INLINE_COPY_TO_USER
static inline __must_check unsigned long
_copy_to_user(void __user *to, const void *from, unsigned long n)
{
	might_fault();
	if (should_fail_usercopy())
		return n;
	if (access_ok(to, n)) {
		instrument_copy_to_user(to, from, n);
		n = raw_copy_to_user(to, from, n);
	}
	return n;
}
#else
extern __must_check unsigned long
_copy_to_user(void __user *, const void *, unsigned long);
#endif
```

[https://elixir.bootlin.com/linux/v6.10.1/source/lib/usercopy.c#L39](https://elixir.bootlin.com/linux/v6.10.1/source/lib/usercopy.c#L39)
```c
#ifndef INLINE_COPY_TO_USER
unsigned long _copy_to_user(void __user *to, const void *from, unsigned long n)
{
	might_fault();
	if (should_fail_usercopy())
		return n;
	if (likely(access_ok(to, n))) {
		instrument_copy_to_user(to, from, n);
		n = raw_copy_to_user(to, from, n);
	}
	return n;
}
EXPORT_SYMBOL(_copy_to_user);
#endif
```

### [`raw_copy_to_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/include/asm/uaccess_64.h#L129)

`copy_user_generic()` のラッパー。

```c
static __always_inline __must_check unsigned long
raw_copy_to_user(void __user *dst, const void *src, unsigned long size)
{
	return copy_user_generic((__force void *)dst, src, size);
}
```

### [`copy_user_generic()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/include/asm/uaccess_64.h#L103)

最初と最後に `stac()` と `clac()` を読んで、ユーザー空間へのアクセスを有効にしている。

FSRM (Fast Short REP MOV) をサポートしている場合は `rep movs` を使い、そうでない場合は [`rep_movs_alternative`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/lib/copy_user_64.S#L32) を使っている。

FSRM が利用可能とここでは想定することにする。

アセンブリの部分をちょっと確認。
- `len` は RCX レジスタにセットされる。
- `to` は DI レジスタにセットされる。
- `from` は SI レジスタにセットされる。

[REP 命令](https://www.felixcloutier.com/x86/rep:repe:repz:repne:repnz) は、RCX = 0 になるまでループする。
[MOVSB 命令](https://www.felixcloutier.com/x86/movs:movsb:movsw:movsd:movsq) は、RSI から RDI へ１バイトコピーする。

```c
static __always_inline __must_check unsigned long
copy_user_generic(void *to, const void *from, unsigned long len)
{
	stac();
	/*
	 * If CPU has FSRM feature, use 'rep movs'.
	 * Otherwise, use rep_movs_alternative.
	 */
	asm volatile(
		"1:\n\t"
		ALTERNATIVE("rep movsb",
			    "call rep_movs_alternative", ALT_NOT(X86_FEATURE_FSRM))
		"2:\n"
		_ASM_EXTABLE_UA(1b, 2b)
		:"+c" (len), "+D" (to), "+S" (from), ASM_CALL_CONSTRAINT
		: : "memory", "rax");
	clac();
	return len;
}
```


## [`copy_from_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/include/linux/uaccess.h#L180)

```c
static __always_inline unsigned long __must_check
copy_from_user(void *to, const void __user *from, unsigned long n)
{
	if (check_copy_size(to, n, false))
		n = _copy_from_user(to, from, n);
	return n;
}
```

### `_copy_from_user()`

`_copy_to_user()` と同様に、インラインバージョンと通常バージョンが定義されている。
インラインバージョンには、speculation barrier がない。

`raw_copy_from_user()` が本体で、もし失敗した場合は `memset()` で 0 にクリーンアップする。

インラインバージョン

[https://elixir.bootlin.com/linux/v6.10.1/source/include/linux/uaccess.h#L143](https://elixir.bootlin.com/linux/v6.10.1/source/include/linux/uaccess.h#L143)
```c
#ifdef INLINE_COPY_FROM_USER
static inline __must_check unsigned long
_copy_from_user(void *to, const void __user *from, unsigned long n)
{
	unsigned long res = n;
	might_fault();
	if (!should_fail_usercopy() && likely(access_ok(from, n))) {
		instrument_copy_from_user_before(to, from, n);
		res = raw_copy_from_user(to, from, n);
		instrument_copy_from_user_after(to, from, n, res);
	}
	if (unlikely(res))
		memset(to + (n - res), 0, res);
	return res;
}
#else
extern __must_check unsigned long
_copy_from_user(void *, const void __user *, unsigned long);
#endif
```

通常バージョン

[https://elixir.bootlin.com/linux/v6.10.1/source/lib/usercopy.c#L16](https://elixir.bootlin.com/linux/v6.10.1/source/lib/usercopy.c#L16)
```c
#ifndef INLINE_COPY_FROM_USER
unsigned long _copy_from_user(void *to, const void __user *from, unsigned long n)
{
	unsigned long res = n;
	might_fault();
	if (!should_fail_usercopy() && likely(access_ok(from, n))) {
		/*
		 * Ensure that bad access_ok() speculation will not
		 * lead to nasty side effects *after* the copy is
		 * finished:
		 */
		barrier_nospec();
		instrument_copy_from_user_before(to, from, n);
		res = raw_copy_from_user(to, from, n);
		instrument_copy_from_user_after(to, from, n, res);
	}
	if (unlikely(res))
		memset(to + (n - res), 0, res);
	return res;
}
EXPORT_SYMBOL(_copy_from_user);
#endif
```

### [`barrier_nospec()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/include/asm/barrier.h#L48)

LFENCE 命令がバリアとして利用されている。

```c
/* Prevent speculative execution past this barrier. */
#define barrier_nospec() alternative("", "lfence", X86_FEATURE_LFENCE_RDTSC)
```

### [`raw_copy_from_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/x86/include/asm/uaccess_64.h#L123)

`raw_copy_to_user()` と同様に `copy_user_generic()` を呼び出す。

```c
static __always_inline __must_check unsigned long
raw_copy_from_user(void *dst, const void __user *src, unsigned long size)
{
	return copy_user_generic(dst, (__force void *)src, size);
}
```


# arm64


## [`put_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L340)

`__put_user` のラッパー。

```c
#define put_user	__put_user
```

### [`__put_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L333)

```c
#define __put_user(x, ptr)						\
({									\
	int __pu_err = 0;						\
	__put_user_error((x), (ptr), __pu_err);				\
	__pu_err;							\
})
```

### [`__put_user_error()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L321)

`access_ok()` でユーザー空間のポインタかチェックした後に、`uaccess_mask_ptr()` でポインタのサニタイズを行う。

```c
#define __put_user_error(x, ptr, err)					\
do {									\
	__typeof__(*(ptr)) __user *__p = (ptr);				\
	might_fault();							\
	if (access_ok(__p, sizeof(*__p))) {				\
		__p = uaccess_mask_ptr(__p);				\
		__raw_put_user((x), __p, (err));			\
	} else	{							\
		(err) = -EFAULT;					\
	}								\
} while (0)
```

### [`uaccess_mask_ptr()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L164)

[BIC 命令](https://developer.arm.com/documentation/ddi0602/2024-06/Base-Instructions/BIC--shifted-register---Bitwise-bit-clear--shifted-register--)は、以下の操作をする命令である。

> BIC <Xd>, <Xn>, <Xm>{, <shift> #<amount>}

> Operation
> ```
> constant bits(datasize) operand1 = X[n, datasize];
> constant bits(datasize) operand2 = ShiftReg(m, shift_type, shift_amount, datasize);
> 
> X[d, datasize] = operand1 AND NOT(operand2);
> ```

以下のパッチに記載の通り、bit 55 がクリアされてれば TTBR0 VA レンジ、bit 55 がセットされてれば TTBR1 VA レンジとなる。

https://lore.kernel.org/all/20220922151053.3520750-1-mark.rutland@arm.com/

> Regardless of the configured VA size or whether TBI is in use, the
> address space can be divided into three ranges:
> 
> * The TTBR0 VA range, for which any valid pointer has bit 55 *clear*,
>   and any non-tag bits [63-56] must match bit 55 (i.e. must be clear).
> 
> * The TTBR1 VA range, for which any valid pointer has bit 55 *set*, and
>   any non-tag bits [63-56] must match bit 55 (i.e. must be set).
> 
> * The gap between the TTBR0 and TTBR1 ranges, where bit 55 may be set or
>   clear, but any access will result in a fault.

[TTBR0 (Translation Table Base Register 0)](https://developer.arm.com/documentation/100442/0100/register-descriptions/aarch32-system-registers/ttbr0--translation-table-base-register-0) と [TTBR1 (Translation Table Base Register 1)](https://developer.arm.com/documentation/100442/0100/register-descriptions/aarch32-system-registers/ttbr1--translation-table-base-register-1) は、Translation Table のベースアドレスを入れるレジスタである。

したがって、`uaccess_mask_ptr()` は bit 55 をクリアすることで、強制的にユーザー空間のアドレスにしている。

```c
/*
 * Sanitize a uaccess pointer such that it cannot reach any kernel address.
 *
 * Clearing bit 55 ensures the pointer cannot address any portion of the TTBR1
 * address range (i.e. any kernel address), and either the pointer falls within
 * the TTBR0 address range or must cause a fault.
 */
#define uaccess_mask_ptr(ptr) (__typeof__(ptr))__uaccess_mask_ptr(ptr)
static inline void __user *__uaccess_mask_ptr(const void __user *ptr)
{
	void __user *safe_ptr;

	asm volatile(
	"	bic	%0, %1, %2\n"
	: "=r" (safe_ptr)
	: "r" (ptr),
	  "i" (BIT(55))
	);

	return safe_ptr;
}
```

### [`__raw_put_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L310)

`uaccess_ttbr0_enable()` でユーザー空間へのアクセスを有効にし、`uaccess_ttbr0_disable()` で無効にする。(詳細は一旦割愛)

```c
/*
 * We must not call into the scheduler between uaccess_ttbr0_enable() and
 * uaccess_ttbr0_disable(). As `x` and `ptr` could contain blocking functions,
 * we must evaluate these outside of the critical section.
 */
#define __raw_put_user(x, ptr, err)					\
do {									\
	__typeof__(*(ptr)) __user *__rpu_ptr = (ptr);			\
	__typeof__(*(ptr)) __rpu_val = (x);				\
	__chk_user_ptr(__rpu_ptr);					\
									\
	uaccess_ttbr0_enable();						\
	__raw_put_mem("sttr", __rpu_val, __rpu_ptr, err, U);		\
	uaccess_ttbr0_disable();					\
} while (0)
```

### [`__raw_put_mem()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L284)

サイズの大きさに応じたアセンブリを呼び出す。

```c
#define __raw_put_mem(str, x, ptr, err, type)					\
do {										\
	__typeof__(*(ptr)) __pu_val = (x);					\
	switch (sizeof(*(ptr))) {						\
	case 1:									\
		__put_mem_asm(str "b", "%w", __pu_val, (ptr), (err), type);	\
		break;								\
	case 2:									\
		__put_mem_asm(str "h", "%w", __pu_val, (ptr), (err), type);	\
		break;								\
	case 4:									\
		__put_mem_asm(str, "%w", __pu_val, (ptr), (err), type);		\
		break;								\
	case 8:									\
		__put_mem_asm(str, "%x", __pu_val, (ptr), (err), type);		\
		break;								\
	default:								\
		BUILD_BUG();							\
	}									\
} while (0)
```

### [`__put_mem_asm()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L276)

```c
#define __put_mem_asm(store, reg, x, addr, err, type)			\
	asm volatile(							\
	"1:	" store "	" reg "1, [%2]\n"			\
	"2:\n"								\
	_ASM_EXTABLE_##type##ACCESS_ERR(1b, 2b, %w0)			\
	: "+r" (err)							\
	: "rZ" (x), "r" (addr))
```


## [`get_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L254)

基本的には `put_user()` と同じ感じ。

```c
#define get_user	__get_user
```

### [`__get_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L247)

```c
#define __get_user(x, ptr)						\
({									\
	int __gu_err = 0;						\
	__get_user_error((x), (ptr), __gu_err);				\
	__gu_err;							\
})
```

### [`__get_user_error()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L235)

```c
#define __get_user_error(x, ptr, err)					\
do {									\
	__typeof__(*(ptr)) __user *__p = (ptr);				\
	might_fault();							\
	if (access_ok(__p, sizeof(*__p))) {				\
		__p = uaccess_mask_ptr(__p);				\
		__raw_get_user((x), __p, (err));			\
	} else {							\
		(x) = (__force __typeof__(x))0; (err) = -EFAULT;	\
	}								\
} while (0)
```

### [`__raw_get_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L222)

```c
/*
 * We must not call into the scheduler between uaccess_ttbr0_enable() and
 * uaccess_ttbr0_disable(). As `x` and `ptr` could contain blocking functions,
 * we must evaluate these outside of the critical section.
 */
#define __raw_get_user(x, ptr, err)					\
do {									\
	__typeof__(*(ptr)) __user *__rgu_ptr = (ptr);			\
	__typeof__(x) __rgu_val;					\
	__chk_user_ptr(ptr);						\
									\
	uaccess_ttbr0_enable();						\
	__raw_get_mem("ldtr", __rgu_val, __rgu_ptr, err, U);		\
	uaccess_ttbr0_disable();					\
									\
	(x) = __rgu_val;						\
} while (0)
```

### [`__raw_get_mem()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L195)

```c
#define __raw_get_mem(ldr, x, ptr, err, type)					\
do {										\
	unsigned long __gu_val;							\
	switch (sizeof(*(ptr))) {						\
	case 1:									\
		__get_mem_asm(ldr "b", "%w", __gu_val, (ptr), (err), type);	\
		break;								\
	case 2:									\
		__get_mem_asm(ldr "h", "%w", __gu_val, (ptr), (err), type);	\
		break;								\
	case 4:									\
		__get_mem_asm(ldr, "%w", __gu_val, (ptr), (err), type);		\
		break;								\
	case 8:									\
		__get_mem_asm(ldr, "%x",  __gu_val, (ptr), (err), type);	\
		break;								\
	default:								\
		BUILD_BUG();							\
	}									\
	(x) = (__force __typeof__(*(ptr)))__gu_val;				\
} while (0)
```

### [`__get_mem_asm()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L187)

```c
/*
 * The "__xxx" versions of the user access functions do not verify the address
 * space - it must have been done previously with a separate "access_ok()"
 * call.
 *
 * The "__xxx_error" versions set the third argument to -EFAULT if an error
 * occurs, and leave it unchanged on success.
 */
#define __get_mem_asm(load, reg, x, addr, err, type)			\
	asm volatile(							\
	"1:	" load "	" reg "1, [%2]\n"			\
	"2:\n"								\
	_ASM_EXTABLE_##type##ACCESS_ERR_ZERO(1b, 2b, %w0, %w1)		\
	: "+r" (err), "=r" (x)						\
	: "r" (addr))
```


## `copy_to_user()`

`copy_to_user()` と `_copy_to_user()` はアーキテクチャ依存しない部分。
`raw_copy_to_user()` がアーキテクチャ依存。

### [`raw_copy_to_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L374)

```c
#define raw_copy_to_user(to, from, n)					\
({									\
	unsigned long __actu_ret;					\
	uaccess_ttbr0_enable();						\
	__actu_ret = __arch_copy_to_user(__uaccess_mask_ptr(to),	\
				    (from), (n));			\
	uaccess_ttbr0_disable();					\
	__actu_ret;							\
})
```

### [`__arch_copy_to_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/lib/copy_to_user.S#L56)

ちょっとこの辺のアセンブリを全部追う元気が残ってない。

```c
end	.req	x5
srcin	.req	x15
SYM_FUNC_START(__arch_copy_to_user)
	add	end, x0, x2
	mov	srcin, x1
#include "copy_template.S"
	mov	x0, #0
	ret

	// Exception fixups
9997:	cmp	dst, dstin
	b.ne	9998f
	// Before being absolutely sure we couldn't copy anything, try harder
	ldrb	tmp1w, [srcin]
USER(9998f, sttrb tmp1w, [dst])
	add	dst, dst, #1
9998:	sub	x0, end, dst			// bytes not copied
	ret
SYM_FUNC_END(__arch_copy_to_user)
EXPORT_SYMBOL(__arch_copy_to_user)
```

### [`copy_template.S`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/lib/copy_template.S)

同様にアセンブリを全部追う元気が残ってない。
ただ、意図してる操作はコメントにある通り。

```c
/*
 * Copy a buffer from src to dest (alignment handled by the hardware)
 *
 * Parameters:
 *	x0 - dest
 *	x1 - src
 *	x2 - n
 * Returns:
 *	x0 - dest
 */
```


## `copy_from_user()`

`copy_from_user()` と `_copy_from_user()` はアーキテクチャ依存しない部分。
`raw_copy_from_user()` がアーキテクチャ依存。

### [`raw_copy_from_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L363)

```c
extern unsigned long __must_check __arch_copy_from_user(void *to, const void __user *from, unsigned long n);
#define raw_copy_from_user(to, from, n)					\
({									\
	unsigned long __acfu_ret;					\
	uaccess_ttbr0_enable();						\
	__acfu_ret = __arch_copy_from_user((to),			\
				      __uaccess_mask_ptr(from), (n));	\
	uaccess_ttbr0_disable();					\
	__acfu_ret;							\
})
```

### [`__arch_copy_from_user()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/lib/copy_from_user.S#L57)

```c
end	.req	x5
srcin	.req	x15
SYM_FUNC_START(__arch_copy_from_user)
	add	end, x0, x2
	mov	srcin, x1
#include "copy_template.S"
	mov	x0, #0				// Nothing to copy
	ret

	// Exception fixups
9997:	cmp	dst, dstin
	b.ne	9998f
	// Before being absolutely sure we couldn't copy anything, try harder
USER(9998f, ldtrb tmp1w, [srcin])
	strb	tmp1w, [dst], #1
9998:	sub	x0, end, dst			// bytes not copied
	ret
SYM_FUNC_END(__arch_copy_from_user)
EXPORT_SYMBOL(__arch_copy_from_user)
```

