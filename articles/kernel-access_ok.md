---
title: 【Kernel】access_ok()
emoji: "🐧"
type: "tech"
topics: ["linux", "kernel"]
published: true
---

※ Linux kernel v6.9.8 のコードベースを元に調査したものです。


# `access_ok()`

https://www.kernel.org/doc/html/v5.0/core-api/mm-api.html

`access_ok(addr, size)` は、与えられたメモリブロックがユーザー空間として有効なものかどうかをチェックする関数である。

引数
- `addr`: ユーザー空間のメモリブロックの開始アドレスへのポインタ
- `size`: メモリブロックのサイズ


# x86 の場合

## `access_ok()`

x86 専用の `access_ok()` は定義されていないので、`include/asm-generic/access_ok.h` で定義されてる一般的なものが使用されれている模様である: https://elixir.bootlin.com/linux/v6.9.8/C/ident/access_ok

具体的には、コードは以下の通りである。

https://elixir.bootlin.com/linux/v6.9.8/source/include/asm-generic/access_ok.h#L45
```c
#ifndef access_ok
#define access_ok(addr, size) likely(__access_ok(addr, size))
#endif
```

`likely()` については、[【Kernel】likely() / unlikely()](https://zenn.dev/zulinx86/articles/kernel-likely_unlikely) を参照。


## `__access_ok()`

以下の 2 つのルールを強制するものである。

1. メモリブロックの開始アドレス (`ptr`) がユーザー空間にある
2. 開始アドレス (`ptr`) にメモリブロックのサイズを足した値が、カーネル空間までオーバーフローしていない

なお、与えられたメモリブロックサイズがページサイズ以下であれば、1 だけを確認すれば大丈夫である。

https://elixir.bootlin.com/linux/v6.9.8/source/arch/x86/include/asm/uaccess_64.h#L92
```c
/*
 * User pointers can have tag bits on x86-64.  This scheme tolerates
 * arbitrary values in those bits rather then masking them off.
 *
 * Enforce two rules:
 * 1. 'ptr' must be in the user half of the address space
 * 2. 'ptr+size' must not overflow into kernel addresses
 *
 * Note that addresses around the sign change are not valid addresses,
 * and will GP-fault even with LAM enabled if the sign bit is set (see
 * "CR3.LAM_SUP" that can narrow the canonicality check if we ever
 * enable it, but not remove it entirely).
 *
 * So the "overflow into kernel addresses" does not imply some sudden
 * exact boundary at the sign bit, and we can allow a lot of slop on the
 * size check.
 *
 * In fact, we could probably remove the size check entirely, since
 * any kernel accesses will be in increasing address order starting
 * at 'ptr', and even if the end might be in kernel space, we'll
 * hit the GP faults for non-canonical accesses before we ever get
 * there.
 *
 * That's a separate optimization, for now just handle the small
 * constant case.
 */
static inline bool __access_ok(const void __user *ptr, unsigned long size)
{
	if (__builtin_constant_p(size <= PAGE_SIZE) && size <= PAGE_SIZE) {
		return valid_user_address(ptr);
	} else {
		unsigned long sum = size + (__force unsigned long)ptr;

		return valid_user_address(sum) && sum >= (__force unsigned long)ptr;
	}
}
#define __access_ok __access_ok
```

## `__builtin_constant_p()`

`int __builtin_constant_p (exp)` は、`exp` がコンパイル時に定数であるとわかっている場合に 1 を、そうではない場合に 0 を返す。

https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#index-_005f_005fbuiltin_005fconstant_005fp

> Built-in Function: `int __builtin_constant_p (exp)`
> 
> You can use the built-in function `__builtin_constant_p` to determine if the expression `exp` is known to be constant at compile time and hence that GCC can perform constant-folding on expressions involving that value. The argument of the function is the expression to test. The expression is not evaluated, side-effects are discarded. The function returns the integer 1 if the argument is known to be a compile-time constant and 0 if it is not known to be a compile-time constant. Any expression that has side-effects makes the function return 0. A return of 0 does not indicate that the expression is not a constant, but merely that GCC cannot prove it is a constant within the constraints of the active set of optimization options.
> 
> You typically use this function in an embedded application where memory is a critical resource. If you have some complex calculation, you may want it to be folded if it involves constants, but need to call a function if it does not. For example:
> 
> ```
> #define Scale_Value(X)      \
>   (__builtin_constant_p (X) \
>   ? ((X) * SCALE + OFFSET) : Scale (X))
> ```
> 
> You may use this built-in function in either a macro or an inline function. However, if you use it in an inlined function and pass an argument of the function as the argument to the built-in, GCC never returns 1 when you call the inline function with a string constant or compound literal (see Compound Literals) and does not return 1 when you pass a constant numeric value to the inline function unless you specify the -O option.
> 
> You may also use `__builtin_constant_p` in initializers for static data. For instance, you can write
> 
> ```
> static const int table[] = {
>    __builtin_constant_p (EXPRESSION) ? (EXPRESSION) : -1,
>    /* … */
> };
> ```
> 
> This is an acceptable initializer even if `EXPRESSION` is not a constant expression, including the case where `__builtin_constant_p` returns `1` because `EXPRESSION` can be folded to a constant but `EXPRESSION` contains operands that are not otherwise permitted in a static initializer (for example, `0 && foo ()`). GCC must be more conservative about evaluating the built-in in this case, because it has no opportunity to perform optimization.

## `valid_user_address()`

仮想アドレス空間は、論理的にカーネル側とユーザー側に分けられており、符号ありの整数型にキャストした場合に、ポインタが整数になればユーザー空間で、負になればカーネル空間にあるということになる。

https://elixir.bootlin.com/linux/v6.9.8/source/arch/x86/include/asm/uaccess_64.h#L54

```c
/*
 * The virtual address space space is logically divided into a kernel
 * half and a user half.  When cast to a signed type, user pointers
 * are positive and kernel pointers are negative.
 */
#define valid_user_address(x) ((__force long)(x) >= 0)
```

https://docs.kernel.org/arch/x86/x86_64/mm.html

上記のカーネルのドキュメントに記載がある通り、メモリ配置は以下の通り。

- 4-level page table の場合
    - 0000000000000000 - 00007fffffffffff: ユーザー空間
    - ffff800000000000 - ffffffffffffffff: カーネル空間
- 5-level page table の場合
    - 0000000000000000 - 00ffffffffffffff: ユーザー空間
    - ff00000000000000 - ffffffffffffffff: カーネル空間

したがって、最上位ビットが 1 になっていれば、カーネル空間ということができ、つまり符号付き整数にした時に負の値になる。
