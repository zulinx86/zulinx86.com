---
title: 【Kernel】Spectre-V1 に対する緩和策
emoji: "🐧"
type: "tech"
topics: ["linux", "kernel"]
published: true
---


[Kernel documentation](https://docs.kernel.org/admin-guide/hw-vuln/spectre.html#spectre-system-information) にある通り、Spectre-V1 に対する Linux カーネルの緩和状況を確認するための sysfs (`/sys/devices/system/cpu/vulnerabilities/spectre_v1`) では、"__user pointer sanitization" と "usercopy barriers" が表示される (ただし、"swapgs barriers" は Spectre-V1 (swapgs) への緩和策) 。

本記事は、これらが具体的にどのように実装されているのか、アーキテクチャごとのコミットの履歴を元に追跡していった際の履歴である。

※ 内容の正確性は保証できません。執筆時点での最新のカーネルバージョン (6.10) を元にしています。

# x86

https://lore.kernel.org/all/151727412964.33451.17213780323040673404.stgit@dwillia2-desk3.amr.corp.intel.com/

上記のパッチセットは、最初に Spectre-V1 に対する x86 向けの緩和策が取り込まれたものである。

パッチセット全体の概要としては、以下の通り。
- 01/13: `array_index_nospec()` に関するドキュメントの追加
- 02/13: `array_index_nospec()` およびアーキテクチャに依存しない `array_index_mask_nospec()` の実装の追加
- 03/13: x86 用の `array_index_mask_nospec()` の実装の追加
- 04/13: `barrier_nospec()` の追加
- 05/13: `barrier_nospec()` を利用した `__uaccess_begin_nospec()` の追加
- 06/13: `stac()`/`clac()` を `__uaccess_begin()`/`__uaccess_end()` に置換 (次のコミットのための準備)
- 07/13: `__uaccess_begin()` を `__uaccess_begin_nospec()` に置換
- 08/13: ポインタのマスキングの適用
- 09/13: `array_index_nospec()` を使ったマスキングを syscall table へのアクセスに適用
- 10/13: `array_index_nospec()` を使ったマスキングをファイルディスクリプタテーブルへのアクセスに適用
- 11/13: `array_inedx_nospec()` を使ったマスキングを VMCS 関連のテーブルアクセスに適用
- 12/13: `array_index_nospec()` を使ったマスキングを Wireless ドライバへ適用
- 13/13: spectre_v1 sysfs の内容の更新

また GitHub 上で[対応するコミット](https://github.com/torvalds/linux/commit/edfbae53dab8348fca778531be9f4855d2ca0360)を見れば、v4.16 に最初にマージされたことがわかる。


## v4.16

### [`array_index_nospec()`](https://elixir.bootlin.com/linux/v4.16/source/include/linux/nospec.h#L47)

配列の境界チェックの後に、配列アクセスに使うインデックスを無害化する関数。

```c
/*
 * array_index_nospec - sanitize an array index after a bounds check
 *
 * For a code sequence like:
 *
 *     if (index < size) {
 *         index = array_index_nospec(index, size);
 *         val = array[index];
 *     }
 *
 * ...if the CPU speculates past the bounds check then
 * array_index_nospec() will clamp the index within the range of [0,
 * size).
 */
#define array_index_nospec(index, size)					\
({									\
	typeof(index) _i = (index);					\
	typeof(size) _s = (size);					\
	unsigned long _mask = array_index_mask_nospec(_i, _s);		\
									\
	BUILD_BUG_ON(sizeof(_i) > sizeof(long));			\
	BUILD_BUG_ON(sizeof(_s) > sizeof(long));			\
									\
	(typeof(_i)) (_i & _mask);					\
})
```

投機的実行で境界チェックがバイパスされてしまった場合に、配列の範囲外にアクセスしてしまわないようにするために、条件分岐命令を使わずに、インデックスが範囲内であれば `~0` でマスクし、範囲外であれば `0` でマスクする。

マスクの生成は `array_index_mask_nospec()` で行われている。

### [`array_index_mask_nospec()`](https://elixir.bootlin.com/linux/v4.16/source/arch/x86/include/asm/barrier.h#L36)

マスクを生成する x86 用の関数。

```c
/**
 * array_index_mask_nospec() - generate a mask that is ~0UL when the
 * 	bounds check succeeds and 0 otherwise
 * @index: array element index
 * @size: number of elements in array
 *
 * Returns:
 *     0 - (index < size)
 */
static inline unsigned long array_index_mask_nospec(unsigned long index,
		unsigned long size)
{
	unsigned long mask;

	asm ("cmp %1,%2; sbb %0,%0;"
			:"=r" (mask)
			:"g"(size),"r" (index)
			:"cc");
	return mask;
}
```

少しややこしいので step by step で解説する。

- [CMP 命令](https://www.felixcloutier.com/x86/cmp)は、２つのオペランドの大きさを比較して、結果の応じて EFLAGS レジスタのステータスフラグをセットする。内部的には [SUB 命令](https://www.felixcloutier.com/x86/sub) と等価である。
- SUB 命令は `DEST := (DEST - SRC)` を行っており、`DEST < SRC` の場合に最上位ビットへボローが発生し、`CF = 1` となる。
- C 言語ではデフォルトで [AT&T 記法](https://en.wikipedia.org/wiki/X86_assembly_language#Syntax)が利用されており、`INSTR SRC,DEST` の形式で記述される。
- `cmp %1,%2` を C の変数名で置き換えると、`cmp size,index` となり、`index - size` と等価になる。つまり、`index < size` の場合に `CF = 1` となる。
- [SBB 命令](https://www.felixcloutier.com/x86/sbb) は `DEST := (DEST - (SRC + CF))` を行う命令である。
- `sbb %0,%0` は、`index < size` の場合に `0 - (0 + 1) = ~0` となり、全てのビットが `1` のマスクが生成される。他方、`index > size` の場合に `0 - (0 + 0) = 0` となり、全てのビットが `0` のマスクになる。

このように条件ジャンプ命令を使わずに、マスクを生成することができる。

### [`barrier_nospec()`](https://elixir.bootlin.com/linux/v4.16/source/arch/x86/include/asm/barrier.h#L52)

投機的実行をバリアするための関数。

```c
/* Prevent speculative execution past this barrier. */
#define barrier_nospec() alternative_2("", "mfence", X86_FEATURE_MFENCE_RDTSC, \
					   "lfence", X86_FEATURE_LFENCE_RDTSC)
```

[MFENCE 命令](https://www.felixcloutier.com/x86/mfence) は、この命令以前のメモリからの読み込み命令とメモリへの書き出し命令が完了するまで待つ命令である。
[LFENCE 命令](https://www.felixcloutier.com/x86/lfence) は、この命令以前のメモリからの読み込み命令が完了するまで待つ命令である。

### [`__uaccess_begin_nospec()`](https://elixir.bootlin.com/linux/v4.16/source/arch/x86/include/asm/uaccess.h#L127)

以下のパッチのメッセージにある通り、`__get_user()` のパスでカーネルがユーザーから渡されたポインタに対して、投機的実行をしてしまわないようにするために、`__uaccess_begin_nospec()` を使って投機的実行が行われる前に `access_ok()` が解決するようにする。なお、`get_user()` に関しては `array_index_nospec()` と同じようにポインタをマスキングすることで対応する。

https://lore.kernel.org/all/151727415922.33451.5796614273104346583.stgit@dwillia2-desk3.amr.corp.intel.com/

> For __get_user() paths, do not allow the kernel to speculate on the
> value of a user controlled pointer. In addition to the 'stac'
> instruction for Supervisor Mode Access Protection (SMAP), a
> barrier_nospec() causes the access_ok() result to resolve in the
> pipeline before the CPU might take any speculative action on the pointer
> value. Given the cost of 'stac' the speculation barrier is placed after
> 'stac' to hopefully overlap the cost of disabling SMAP with the cost of
> flushing the instruction pipeline.
> 
> Since __get_user is a major kernel interface that deals with user
> controlled pointers, the __uaccess_begin_nospec() mechanism will prevent
> speculative execution past an access_ok() permission check. While
> speculative execution past access_ok() is not enough to lead to a kernel
> memory leak, it is a necessary precondition.
> 
> To be clear, __uaccess_begin_nospec() is addressing a class of potential
> problems near __get_user() usages.
> 
> Note, that while the barrier_nospec() in __uaccess_begin_nospec() is
> used to protect __get_user(), pointer masking similar to
> array_index_nospec() will be used for get_user() since it incorporates a
> bounds check near the usage.

https://lore.kernel.org/all/151727416953.33451.10508284228526170604.stgit@dwillia2-desk3.amr.corp.intel.com/

> __uaccess_begin_nospec() covers __get_user() and copy_from_iter() where
> the limit check is far away from the user pointer de-reference. In those
> cases a barrier_nospec() prevents speculation with a potential pointer to
> privileged memory.

実装としては [STAC 命令](https://www.felixcloutier.com/x86/stac)の後に `barrier_nospec()` を呼び出している。

```c
#define __uaccess_begin_nospec()	\
({					\
	stac();				\
	barrier_nospec();		\
})
```


### [`__get_user()`](https://elixir.bootlin.com/linux/v4.16/source/arch/x86/include/asm/uaccess.h#L526)

`__get_user()` のコメントにある通り、`access_ok()` を呼び出さなければならない。

実装的には `__get_user_nocheck()` を呼び出しているだけである。

```c
/**
 * __get_user: - Get a simple variable from user space, with less checking.
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
 * Caller must check the pointer with access_ok() before calling this
 * function.
 *
 * Returns zero on success, or -EFAULT on error.
 * On error, the variable @x is set to zero.
 */

#define __get_user(x, ptr)						\
	__get_user_nocheck((x), (ptr), sizeof(*(ptr)))
```

### [`__get_user_nocheck()`](https://elixir.bootlin.com/linux/v4.16/source/arch/x86/include/asm/uaccess.h#L449)

確かに `__get_user_size()` で実際にアクセスする前に、`__uaccess_begin_nospec()` が入っている。

```c
#define __get_user_nocheck(x, ptr, size)				\
({									\
	int __gu_err;							\
	__inttype(*(ptr)) __gu_val;					\
	__uaccess_begin_nospec();					\
	__get_user_size(__gu_val, (ptr), (size), __gu_err, -EFAULT);	\
	__uaccess_end();						\
	(x) = (__force __typeof__(*(ptr)))__gu_val;			\
	__builtin_expect(__gu_err, 0);					\
})
```


### [`get_user()`](https://elixir.bootlin.com/linux/v4.16/source/arch/x86/include/asm/uaccess.h#L449)

以下のパッチのメッセージにある通り、`get_user()` では、ポインタの参照外しがアドレスの制限チェックの近くにあるので、投機的実行バリア (`barrier_nospec()`) の代わりに、ポインタの無害化をマスクを使って行う (`array_index_nospec()`) 。

https://lore.kernel.org/all/151727417469.33451.11804043010080838495.stgit@dwillia2-desk3.amr.corp.intel.com/

> Unlike the __get_user() case get_user() includes the address limit check
> near the pointer de-reference. With that locality the speculation can be
> mitigated with pointer narrowing rather than a barrier, i.e.
> array_index_nospec(). Where the narrowing is performed by:
> 
> 	cmp %limit, %ptr
> 	sbb %mask, %mask
> 	and %mask, %ptr
> 
> With respect to speculation the value of %ptr is either less than %limit
> or NULL.

`__get_user()` と異なって、事前に `access_ok()` を呼び出さなければならないという制約がない模様。

```c
/**
 * get_user: - Get a simple variable from user space.
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
 * Returns zero on success, or -EFAULT on error.
 * On error, the variable @x is set to zero.
 */
/*
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
#define get_user(x, ptr)						\
({									\
	int __ret_gu;							\
	register __inttype(*(ptr)) __val_gu asm("%"_ASM_DX);		\
	__chk_user_ptr(ptr);						\
	might_fault();							\
	asm volatile("call __get_user_%P4"				\
		     : "=a" (__ret_gu), "=r" (__val_gu),		\
			ASM_CALL_CONSTRAINT				\
		     : "0" (ptr), "i" (sizeof(*(ptr))));		\
	(x) = (__force __typeof__(*(ptr))) __val_gu;			\
	__builtin_expect(__ret_gu, 0);					\
})
```

### [`__get_user_<size>()`](https://elixir.bootlin.com/linux/v4.16/source/arch/x86/lib/getuser.S#L39)

```s
ENTRY(__get_user_1)
	mov PER_CPU_VAR(current_task), %_ASM_DX
	cmp TASK_addr_limit(%_ASM_DX),%_ASM_AX
	jae bad_get_user
	sbb %_ASM_DX, %_ASM_DX		/* array_index_mask_nospec() */
	and %_ASM_DX, %_ASM_AX
	ASM_STAC
1:	movzbl (%_ASM_AX),%edx
	xor %eax,%eax
	ASM_CLAC
	ret
ENDPROC(__get_user_1)
EXPORT_SYMBOL(__get_user_1)


ENTRY(__get_user_2)
	add $1,%_ASM_AX
	jc bad_get_user
	mov PER_CPU_VAR(current_task), %_ASM_DX
	cmp TASK_addr_limit(%_ASM_DX),%_ASM_AX
	jae bad_get_user
	sbb %_ASM_DX, %_ASM_DX		/* array_index_mask_nospec() */
	and %_ASM_DX, %_ASM_AX
	ASM_STAC
2:	movzwl -1(%_ASM_AX),%edx
	xor %eax,%eax
	ASM_CLAC
	ret
ENDPROC(__get_user_2)
EXPORT_SYMBOL(__get_user_2)

ENTRY(__get_user_4)
	add $3,%_ASM_AX
	jc bad_get_user
	mov PER_CPU_VAR(current_task), %_ASM_DX
	cmp TASK_addr_limit(%_ASM_DX),%_ASM_AX
	jae bad_get_user
	sbb %_ASM_DX, %_ASM_DX		/* array_index_mask_nospec() */
	and %_ASM_DX, %_ASM_AX
	ASM_STAC
3:	movl -3(%_ASM_AX),%edx
	xor %eax,%eax
	ASM_CLAC
	ret
ENDPROC(__get_user_4)
EXPORT_SYMBOL(__get_user_4)

ENTRY(__get_user_8)
#ifdef CONFIG_X86_64
	add $7,%_ASM_AX
	jc bad_get_user
	mov PER_CPU_VAR(current_task), %_ASM_DX
	cmp TASK_addr_limit(%_ASM_DX),%_ASM_AX
	jae bad_get_user
	sbb %_ASM_DX, %_ASM_DX		/* array_index_mask_nospec() */
	and %_ASM_DX, %_ASM_AX
	ASM_STAC
4:	movq -7(%_ASM_AX),%rdx
	xor %eax,%eax
	ASM_CLAC
	ret
#else
	add $7,%_ASM_AX
	jc bad_get_user_8
	mov PER_CPU_VAR(current_task), %_ASM_DX
	cmp TASK_addr_limit(%_ASM_DX),%_ASM_AX
	jae bad_get_user_8
	sbb %_ASM_DX, %_ASM_DX		/* array_index_mask_nospec() */
	and %_ASM_DX, %_ASM_AX
	ASM_STAC
4:	movl -7(%_ASM_AX),%edx
5:	movl -3(%_ASM_AX),%ecx
	xor %eax,%eax
	ASM_CLAC
	ret
#endif
ENDPROC(__get_user_8)
EXPORT_SYMBOL(__get_user_8)


bad_get_user:
	xor %edx,%edx
	mov $(-EFAULT),%_ASM_AX
	ASM_CLAC
	ret
END(bad_get_user)
```


## v6.10

### [`array_index_nospec()`](https://elixir.bootlin.com/linux/v4.16/source/arch/x86/lib/getuser.S#L39)

実装は変わってない。

### [`array_index_mask_nospec()`](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/include/asm/barrier.h#L36)

基本的な実装は変わっていない。

### [`barrier_nospec()`](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/include/asm/barrier.h#L48)

```c
/* Prevent speculative execution past this barrier. */
#define barrier_nospec() alternative("", "lfence", X86_FEATURE_LFENCE_RDTSC)
```

MFENCE は[このコミット](https://github.com/torvalds/linux/commit/be261ffce6f13229dad50f59c5e491f933d3167f)で外されている。

### [`__uaccess_begin_nospec()`](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/include/asm/uaccess.h#L39)

基本的な実装は変わっていない。

### [`__get_user()`](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/include/asm/uaccess.h#L131)

`__get_user_nocheck()` を呼び出す代わりに、`do_get_user_call()` が呼びされている。

```c
#define __get_user(x,ptr) do_get_user_call(get_user_nocheck,x,ptr)
```

### [`do_get_user_call()`](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/include/asm/uaccess.h#L76)

内部的には `__get_user_nocheck_<size>()` を呼び出している。

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

### [`__get_user_nocheck_<size>()`](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/lib/getuser.S#L104)

ユーザーポインタにアクセスする前に、`ASM_BARRIER_NOSPEC` を呼び出している。

```s
/* .. and the same for __get_user, just without the range checks */
SYM_FUNC_START(__get_user_nocheck_1)
	ASM_STAC
	ASM_BARRIER_NOSPEC
6:	movzbl (%_ASM_AX),%edx
	xor %eax,%eax
	ASM_CLAC
	RET
SYM_FUNC_END(__get_user_nocheck_1)
EXPORT_SYMBOL(__get_user_nocheck_1)

SYM_FUNC_START(__get_user_nocheck_2)
	ASM_STAC
	ASM_BARRIER_NOSPEC
7:	movzwl (%_ASM_AX),%edx
	xor %eax,%eax
	ASM_CLAC
	RET
SYM_FUNC_END(__get_user_nocheck_2)
EXPORT_SYMBOL(__get_user_nocheck_2)

SYM_FUNC_START(__get_user_nocheck_4)
	ASM_STAC
	ASM_BARRIER_NOSPEC
8:	movl (%_ASM_AX),%edx
	xor %eax,%eax
	ASM_CLAC
	RET
SYM_FUNC_END(__get_user_nocheck_4)
EXPORT_SYMBOL(__get_user_nocheck_4)

SYM_FUNC_START(__get_user_nocheck_8)
	ASM_STAC
	ASM_BARRIER_NOSPEC
#ifdef CONFIG_X86_64
9:	movq (%_ASM_AX),%rdx
#else
9:	movl (%_ASM_AX),%edx
10:	movl 4(%_ASM_AX),%ecx
#endif
	xor %eax,%eax
	ASM_CLAC
	RET
SYM_FUNC_END(__get_user_nocheck_8)
EXPORT_SYMBOL(__get_user_nocheck_8)
```

```c
#define ASM_BARRIER_NOSPEC ALTERNATIVE "", "lfence", X86_FEATURE_LFENCE_RDTSC
```

### [`get_user()`](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/include/asm/uaccess.h#L108)

`__get_user()` と同様に `do_get_user_call()` を呼び出しており、`__get_user_<size>()` が呼び出されることになる。

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

### [`__get_user_<size>()`](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/lib/getuser.S#L58)

`check_range` が呼び出されている。

```s
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

### [`check_range`](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/lib/getuser.S#L40)

64-bit では、最上位ビットが 1 かどうかでマスキングする方法に変わっている。

```s
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



# arm64

https://lore.kernel.org/all/1517844864-15887-1-git-send-email-will.deacon@arm.com/#t

パッチセット全体の概要としては、以下の通り。
- 1/9: 投機的実行バリアとして `csdb` マクロの追加
- 2/9: arm64 用の `array_index_mask_nospec()` の実装
- 3/9: `USER_DS` を inclusive limit にする (以降のコミットのための準備)
- 4/9: ユーザーポインタをマスキングする `__uaccess_mask_ptr()` の実装
- 5/9: `mask_nospec64` アセンブリの実装と syscall table への適用
- 6/9: 誤った条件分岐予測によって呼び出された `set_fs()` がアドレス制限を設定し、その後の `access_ok()` に渡されないようにする
- 7/9: `__get_user()` / `__put_User()` でも `access_ok()` と `uaccess_mask_ptr()` を呼ぶように変更
- 8/9: `__uaccess_mask_ptr()` の適用箇所をさらに拡大
- 9/9: futex へユーザーポインタマスキングの適用

また GitHub 上で[対応するコミット](https://github.com/torvalds/linux/commit/669474e772b952b14f4de4845a1558fd4c0414a4)を見れば、v4.16 に最初にマージされたことがわかる。


## v4.16

### [`csdb()`](https://elixir.bootlin.com/linux/v4.16/source/arch/arm64/include/asm/barrier.h#L35)

```c
#define csdb()		asm volatile("hint #20" : : : "memory")
```

アセンブリバージョンも実装されている。

[https://elixir.bootlin.com/linux/v4.16/source/arch/arm64/include/asm/assembler.h#L121](https://elixir.bootlin.com/linux/v4.16/source/arch/arm64/include/asm/assembler.h#L121)
```s
/*
 * Value prediction barrier
 */
	.macro	csdb
	hint	#20
	.endm
```

以下のドキュメントにある通り、CSDB 命令は分岐命令自体は投機的実行を許可するが、その後の投機的な実行結果に基づいた命令の投機的実行は許可していない。

https://developer.arm.com/documentation/100076/0100/A32-T32-Instruction-Set-Reference/A32-and-T32-Instructions/CSDB

> Usage
> 
> Consumption of Speculative Data Barrier is a memory barrier that controls Speculative execution and data value prediction. ...
> 
> The `CSDB` instruction allows Speculative execution of:
> - Branch instructions.
> - Instructions that write to the PC.
> - Instructions that are not a result of data value predictions.
> - Instructions that are the result of PSTATE.{N,Z,C,V} predictions from conditional branch instructions or from conditional instructions that write to the PC.
> 
> The `CSDB` instruction prevents Speculative execution of:
> - Non-branch instructions.
> - Instructions that do not write to the PC.
> - Instructions that are the result of data value predictions.
> - Instructions that are the result of PSTATE.{N,Z,C,V} predictions from instructions other than conditional branch instructions and conditional instructions that write to the PC.

また、以下のドキュメントにある通り、CSDB 命令は conditional select または conditional move 命令と組み合わせることで、緩和策として機能する模様。

https://developer.arm.com/documentation/102816/latest/

> Software Mitigations
> 
> The practical software mitigation for the scenario where the value being leaked is determined by less privileged software is to ensure that the address that is derived from the untrusted_offset is forced to be a safe value in a way that the hardware cannot speculate past if the untrusted_offset is out of range.
> 
> This can be achieved on Arm implementations by using a performance optimized code sequence that mitigates speculation and enforces validation of the limits of the untrusted value. Such code sequences are based around specific data processing operations (for example conditional select or conditional move) and a new barrier instruction (CSDB). The combination of both a conditional select/conditional move and the new barrier are sufficient to address this problem on ALL Arm implementations, both current and future.

> Use of the Barrier
> 
> These examples show how we expect the barrier to be used in the assembly code executed on the processor.
> 
> The CSDB instruction prevents an implementation from using hardware data value prediction to speculative the result of a conditional select.
> 
> Taking the example shown previously:
> 
> ```c
> struct array {
>     unsigned long length;
>     unsigned char data[];
> };
> struct array *arr1 = ...; /* small array */
> struct array *arr2 = ...; /* array of size 0x400 */
> unsigned long untrusted_offset_from_user = ...;
> if (untrusted_offset_from_user < arr1->length) {
>     unsigned char value;
>     value = arr1->data[untrusted_offset_from_user];
>     unsigned long index2 = ((value & 1) * 0x100) + 0x200;
>     if (index2 < arr2->length) {
>         unsigned char value2 = arr2->data[index2];
>     }
> }
> ```
> 
> This example would typically be compiled into assembly of the following (simplified) form in AArch64:
> 
> ```s
>     LDR X1, [X2] ; X2 is a pointer to arr1->length
>     CMP X0, X1 ; X0 holds untrusted_offset_from_user
>     BGE out_of_range
>     LDRB W4, [X5, X0] ; X5 holds arr1->data base
>     AND X4, X4, #1
>     LDL X4, X4, #8
>     ADD X4, X4, #0x200
>     CMP X4, X6 ; X6 holds arr2->length
>     BGE out_of_range
>     LDRB X7, [X8, X4] ; X8 holds arr2->data base
> out_of_rage
> ```
> 
> The side-channel can be mitigated in this case by changing this code to be:
> 
> ```s
>     LDR X1, [X2] ; X2 is a pointer to arr1->length
>     CMP X0, X1 ; X0 holds untrusted_offset_from_user
>     BGE out_of_range
>     CSEL X0, XZR, X0, GE
>     CSDB ; this is the new barrier
>     LDRB W4, [X5, X0] ; X5 holds arr1->data base
>     AND X4, X4, #1
>     LDL X4, X4, #8
>     ADD X4, X4, #0x200
>     CMP X4, X6 ; X6 holds arr2->length
>     BGE out_of_range
>     LDRB X7, [X8, X4] ; X8 holds arr2->data base
> out_of_rage
> ```

### [`array_index_mask_nospec()`](https://elixir.bootlin.com/linux/v4.16/source/arch/arm64/include/asm/barrier.h#L48)

x86 と同様に、キャリーを考慮した減算である [SBC 命令](https://developer.arm.com/documentation/ddi0596/2020-12/Base-Instructions/SBC--Subtract-with-Carry-)を使うことで、マスクを生成している。

x86 と違うのは、`array_index_mask_nospec()` の最後に投機的実行バリアである CSDB 命令もおいている点である。
上述の通り、CSDB 命令は conditional select または conditional move と組み合わせて使うことで、バリアとして機能するはずだが、SBC 命令どうなのだろうか。。。

```c
/*
 * Generate a mask for array_index__nospec() that is ~0UL when 0 <= idx < sz
 * and 0 otherwise.
 */
#define array_index_mask_nospec array_index_mask_nospec
static inline unsigned long array_index_mask_nospec(unsigned long idx,
						    unsigned long sz)
{
	unsigned long mask;

	asm volatile(
	"	cmp	%1, %2\n"
	"	sbc	%0, xzr, xzr\n"
	: "=r" (mask)
	: "r" (idx), "Ir" (sz)
	: "cc");

	csdb();
	return mask;
}
```

### [`__uaccess_mask_ptr()`](https://elixir.bootlin.com/linux/v4.16/source/arch/arm64/include/asm/uaccess.h#L241)

ユーザー空間から与えられたポインタのマスキングをする。

こちらは CSDB 命令の前に CSEL 命令があり、投機的実行バリアとしてちゃんと機能してそうである。

```c
/*
 * Sanitise a uaccess pointer such that it becomes NULL if above the
 * current addr_limit.
 */
#define uaccess_mask_ptr(ptr) (__typeof__(ptr))__uaccess_mask_ptr(ptr)
static inline void __user *__uaccess_mask_ptr(const void __user *ptr)
{
	void __user *safe_ptr;

	asm volatile(
	"	bics	xzr, %1, %2\n"
	"	csel	%0, %1, xzr, eq\n"
	: "=&r" (safe_ptr)
	: "r" (ptr), "r" (current_thread_info()->addr_limit)
	: "cc");

	csdb();
	return safe_ptr;
}
```

[BICS 命令](https://developer.arm.com/documentation/ddi0602/2024-06/Base-Instructions/BICS--shifted-register---Bitwise-bit-clear--shifted-register---setting-flags-) は、以下の通り。

> 32-bit (sf == 0)
> BICS <Wd>,<Wn>,<Wm>{,<shift> #<amount>}
> 
> 64-bit (sf == 1)
> BICS <Xd>,<Xn>,<Xm>{,<shift> #<amount>}

> Operation
> ```
> constant bits(datasize) operand1 = X[n, datasize];
> constant bits(datasize) operand2 = ShiftReg(m, shift_type, shift_amount, datasize);
> 
> constant bits(datasize) result = operand1 AND NOT(operand2);
> X[d, datasize] = result;
> PSTATE.<N,Z,C,V> = result<datasize-1>:IsZeroBit(result):'00';
> ```

`bics xzr, %1, %2` を C の変数で置き換えると `bics xzr, ptr, addr_limit` となる。
つまり、`xzr = ptr & !addr_limit` を実行し、以下のようにフラグをセットする。
- `ptr > addr_limit` の場合、結果は非ゼロになり `PSTATE.Z = 0` となる。
- `ptr <= addr_limit` の場合、結果はゼロになり `PSTATE.Z = 1` となる。

[CSEL 命令](https://developer.arm.com/documentation/ddi0602/2024-06/Base-Instructions/CSEL--Conditional-select-) は、以下の通り。

> 32-bit (sf == 0)
> CSEL <Wd>,<Wn>,<Wm>,<cond>
> 
> 64-bit (sf == 1)
> CSEL <Xd>,<Xn>,<Xm>,<cond>

> Operation
> ```
> bits(datasize) result;
> if ConditionHolds(condition) then
>     result = X[n, datasize];
> else
>     result = X[m, datasize];
> 
> X[d, datasize] = result;
> ```

`csel %0, %1, xzr, eq` を C の変数で置き換えると `csel safe_ptr, ptr, xzr, eq` となる。
- `PSTATE.Z = 0` (`ptr > addr_limit`) の場合、`safe_ptr = xzr` となる。
- `PSTATE.Z = 1` (`ptr < addr_limit`) の場合、`safe_ptr = ptr` となる。

### [`raw_copy_from_user()` / `raw_copy_to_user()`](https://elixir.bootlin.com/linux/v4.16/source/arch/arm64/include/asm/uaccess.h#L406)

`__uaccess_mask_ptr()` が使用されている箇所は複数あるが、重要なものとしては `raw_copy_from_user()` と `raw_copy_to_user()` がある。

```c
#define raw_copy_from_user(to, from, n)					\
({									\
	__arch_copy_from_user((to), __uaccess_mask_ptr(from), (n));	\
})
```
```c
#define raw_copy_to_user(to, from, n)					\
({									\
	__arch_copy_to_user(__uaccess_mask_ptr(to), (from), (n));	\
})
```

### [`mask_nospec64`](https://elixir.bootlin.com/linux/v4.16/source/arch/arm64/include/asm/assembler.h#L129)

```s
/*
 * Sanitise a 64-bit bounded index wrt speculation, returning zero if out
 * of bounds.
 */
	.macro	mask_nospec64, idx, limit, tmp
	sub	\tmp, \idx, \limit
	bic	\tmp, \tmp, \idx
	and	\idx, \idx, \tmp, asr #63
	csdb
	.endm
```


[SUB 命令](https://developer.arm.com/documentation/ddi0602/2024-06/Base-Instructions/SUB--shifted-register---Subtract--shifted-register--)

> SUB <Xd>,<Xn>,<Xm>{,<shift> #<amount>}

> ```
> constant bits(datasize) operand1 = X[n, datasize];
> constant bits(datasize) operand2 = NOT(ShiftReg(m, shift_type, shift_amount, datasize));
> bits(datasize) result;
> 
> (result, -) = AddWithCarry(operand1, operand2, '1');
> 
> X[d, datasize] = result;
> ```

[BIC 命令](https://developer.arm.com/documentation/ddi0602/2024-06/Base-Instructions/BIC--shifted-register---Bitwise-bit-clear--shifted-register--)

> BIC <Xd>,<Xn>,<Xm>{,<shift> #<amount>}

> ```
> constant bits(datasize) operand1 = X[n, datasize];
> constant bits(datasize) operand2 = ShiftReg(m, shift_type, shift_amount, datasize);
> 
> X[d, datasize] = operand1 AND NOT(operand2);
> ```

[AND 命令](https://developer.arm.com/documentation/ddi0602/2024-06/Base-Instructions/AND--shifted-register---Bitwise-AND--shifted-register--)

> AND <Xd>,<Xn>,<Xm>{,<shift> #<amount>}

> ```
> constant bits(datasize) operand1 = X[n, datasize];
> constant bits(datasize) operand2 = ShiftReg(m, shift_type, shift_amount, datasize);
> 
> X[d, datasize] = operand1 AND operand2;
> ```

[ShiftReg](https://developer.arm.com/documentation/ddi0602/2024-06/Shared-Pseudocode/aarch64-functions-shiftreg?lang=en#impl-aarch64.ShiftReg.4)

> ```
> bits(N) ShiftReg(integer reg, ShiftType shiftType, integer amount, integer N)
>     bits(N) result = X[reg, N];
>     case shifttype of
>         when ShiftType_LSL result = LSL(result, amount);
>         when ShiftType_LSR result = LSR(result, amount);
>         when ShiftType_ASR result = ASR(result, amount);
>         when ShiftType_ROR result = ROR(result, amount);
>     return result;
> ```

[ASR](https://developer.arm.com/documentation/ddi0602/2024-06/Shared-Pseudocode/shared-functions-common?lang=en#impl-shared.ASR.2)

> ```
> bits(N) ASR(bits(N) x, integer shift)
>     assert shift >= 0
>     bits(N) result;
>     if shift == 0 then
>         result = x;
>     else
>         (result, -) = ASR_C(x, shift);
>     return result;
> ```

[ASR_C](https://developer.arm.com/documentation/ddi0602/2024-06/Shared-Pseudocode/shared-functions-common?lang=en#impl-shared.ASR_C.2)

> ```
> (bits(N), bit) ASR_C(bits(N) x, integer shift)
>     assert shift > 0 && shift < 256
>     extended_x = SignExtend(x, shift+N);
>     result = extended_x<(shift+N)-1:shift>;
>     carry_out = extended_x<shift-1>;
>     return (result, carry_out);
> ```

[SignExtend](https://developer.arm.com/documentation/ddi0602/2024-06/Shared-Pseudocode/shared-functions-common?lang=en#impl-shared.SignExtend.2)

> ```
> bits(N) SignExtend(bits(M) x, integer N)
>     assert N >= M;
>     return Replicate(x<M-1>, N-M) : x;
> ```

[Replicate](https://developer.arm.com/documentation/ddi0602/2024-06/Shared-Pseudocode/shared-functions-common?lang=en#impl-shared.Replicate.2)

> ```
> bits(M*N) Replicate(bits(M) x, integer N);
> ```

CSDB 命令より前の部分の疑似コードとしては、以下の通り。

```c
tmp = idx + ~limit + 1;
tmp = tmp & ~idx;
if (tmp & (1 << (size - 1)))
    idx = idx & ((1 << size) - 1);
else
    idx = idx & 0;
```

C++ で簡易版を実装すると、以下のような感じ。
マスクを作成する時に使うのは、`tmp` の最上位ビット (MSB: Most Significant Bit) だけなので、なぜ BIC 命令が必要なのかは正直わからない。。。

```cpp
using namespace std;

#include <iostream>
#include <bitset>

uint8_t mask_nospec8(uint8_t idx, uint8_t limit) {
    cout << "mask_nospec8(idx=" << (int)idx << ", limit" << (int)limit << ")" << endl;
    cout << "      idx: " << bitset<8>(idx) << endl;
    cout << "    limit: " << bitset<8>(limit) << endl;

    uint8_t tmp = idx + ~limit + 1;
    cout << "  tmp = idx + ~limit + 1;" << endl;
    cout << "      tmp: " << bitset<8>(tmp) << endl;

    tmp = tmp & ~idx;
    cout << "  tmp = tmp & ~idx;" << endl;
    cout << "      tmp: " << bitset<8>(tmp) << endl;

    cout << "      MSB: " << ((tmp & ((uint8_t)1 << (8 - 1))) ? 1 : 0) << endl;
    if (tmp & ((uint8_t)1 << (8 - 1)))
        idx = idx & (((uint16_t)1 << 8) - 1);
    else
        idx = idx & 0;
    cout << "  masking..." << endl;
    cout << "      idx: " << bitset<8>(idx) << endl << endl;
    return idx;
}

int main() {
    cout << "===== OK cases =====" << endl;
    mask_nospec8(1, 3);
    mask_nospec8(10, 100);
    mask_nospec8(126, 127);

    cout << "===== NG cases =====" << endl;
    mask_nospec8(3, 1);
    mask_nospec8(100, 10);
    mask_nospec8(127, 127);
    return 0;
}
```
```
===== OK cases =====
mask_nospec8(idx=1, limit=3)
      idx: 00000001
    limit: 00000011
  tmp = idx + ~limit + 1;
      tmp: 11111110
  tmp = tmp & ~idx;
      tmp: 11111110
      MSB: 1
  masking...
      idx: 00000001

mask_nospec8(idx=10, limit=100)
      idx: 00001010
    limit: 01100100
  tmp = idx + ~limit + 1;
      tmp: 10100110
  tmp = tmp & ~idx;
      tmp: 10100100
      MSB: 1
  masking...
      idx: 00001010

mask_nospec8(idx=126, limit=127)
      idx: 01111110
    limit: 01111111
  tmp = idx + ~limit + 1;
      tmp: 11111111
  tmp = tmp & ~idx;
      tmp: 10000001
      MSB: 1
  masking...
      idx: 01111110

===== NG cases =====
mask_nospec8(idx=3, limit=1)
      idx: 00000011
    limit: 00000001
  tmp = idx + ~limit + 1;
      tmp: 00000010
  tmp = tmp & ~idx;
      tmp: 00000000
      MSB: 0
  masking...
      idx: 00000000

mask_nospec8(idx=100, limit=10)
      idx: 01100100
    limit: 00001010
  tmp = idx + ~limit + 1;
      tmp: 01011010
  tmp = tmp & ~idx;
      tmp: 00011010
      MSB: 0
  masking...
      idx: 00000000

mask_nospec8(idx=127, limit=127)
      idx: 01111111
    limit: 01111111
  tmp = idx + ~limit + 1;
      tmp: 00000000
  tmp = tmp & ~idx;
      tmp: 00000000
      MSB: 0
  masking...
      idx: 00000000
```


## v6.10

### [`csdb()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/barrier.h#L33)

実装に変化なし。

### [`array_index_mask_nospec()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/barrier.h#L87)

```c
/*
 * Generate a mask for array_index__nospec() that is ~0UL when 0 <= idx < sz
 * and 0 otherwise.
 */
#define array_index_mask_nospec array_index_mask_nospec
static inline unsigned long array_index_mask_nospec(unsigned long idx,
						    unsigned long sz)
{
	unsigned long mask;

	asm volatile(
	"	cmp	%1, %2\n"
	"	sbc	%0, xzr, xzr\n"
	: "=r" (mask)
	: "r" (idx), "Ir" (sz)
	: "cc");

	csdb();
	return mask;
}
```

### [`__uaccess_mask_ptr()`](https://elixir.bootlin.com/linux/v6.10.1/source/arch/arm64/include/asm/uaccess.h#L165)

かなり実装が変化していて、CSDB 命令もなくなっている。

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

以下のパッチで更新されている。以前は uaccess プリミティブはカーネルメモリへのアクセスにも使われており、`thread_info::addr_limit` を利用して動的にアドレスマスキングする必要があった。その後に uaccess プリミティブはユーザーメモリへのアクセスにのみ使われるように更新され、アドレス制限はコンパイル時の定数となった。

https://lore.kernel.org/all/20220922151053.3520750-1-mark.rutland@arm.com/

> We introduced uaccess pointer masking for arm64 in commit:
> 
>   4d8efc2d5ee4c9cc ("arm64: Use pointer masking to limit uaccess speculation")
> 
> Which was intended to prevent speculative uaccesses to kernel memory on
> CPUs where access permissions were not respected under speculation.
> 
> At the time, the uaccess primitives were occasionally used to access
> kernel memory, with the maximum permitted address held in
> thread_info::addr_limit. Consequently, the address masking needed to
> take this dynamic limit into account.
>
> Subsequently the uaccess primitives were reworked such that they are
> only used for user memory, and as of commit:
> 
>   3d2403fd10a1dbb3 ("arm64: uaccess: remove set_fs()")
> 
> ... the address limit was made a compile-time constant, but the logic
> was otherwise unchanged.

### [`mask_nospec64`](https://elixir.bootlin.com/linux/v6.10.1/A/ident/mask_nospec64)

v6.10 ではもうなくなっている。

[このコミット](https://github.com/torvalds/linux/commit/4141c857fd09dbed480f021b3eece4f46c653161) でシステムコールは C コードに変換されており、結果的に `mask_nospec64` は使用されなくなった。`invoke_syscall()` では、代わりに `array_index_nospec()` が使用されている。

```c
asmlinkage void invoke_syscall(struct pt_regs *regs, unsigned int scno,
			       unsigned int sc_nr,
			       const syscall_fn_t syscall_table[])
{
	long ret;

	if (scno < sc_nr) {
		syscall_fn_t syscall_fn;
		syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];
		ret = __invoke_syscall(regs, syscall_fn);
	} else {
		ret = do_ni_syscall(regs);
	}

	regs->regs[0] = ret;
}
```

その後、[このコミット](https://github.com/torvalds/linux/commit/c87857945b0e61c46222798b56fb1b0f4868088b) で `mask_nospec64` は消されている。

