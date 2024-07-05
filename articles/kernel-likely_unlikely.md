---
title: 【Kernel】likely() / unlikely()
emoji: "🐧"
type: "tech"
topics: ["linux", "kernel"]
published: true
---

※ Linux kernel v6.9.8 のコードベースを元に調査したものです。


# 分岐予測 (Branch Prediction)

最近の CPU では、[命令パイプライン](https://ja.wikipedia.org/wiki/%E5%91%BD%E4%BB%A4%E3%83%91%E3%82%A4%E3%83%97%E3%83%A9%E3%82%A4%E3%83%B3)が採用されており、1 つの CPU 命令であっても、複数のステージに分けて実行される。

例えば [Classic RISC pipeline](https://en.wikipedia.org/wiki/Classic_RISC_pipeline) では、以下のように 5 つのフェーズに分けられて実行される。
1. 命令フェッチ (IF: Instruction Fetch)
2. 命令でコード (ID: Instruction Decode)
3. 実行 (EX: Execute) 
4. メモリアクセス (MEM: Memory Access)
5. ライトバック (WB: Write-Back)

`if` 文などの分岐命令は、動的にプログラムの実行フローを制御するために使用されるが、それ以前に行われた計算結果が分岐の条件になる。つまり、分岐条件となる計算結果が得られるまでは、次にどの命令を実行すればいいかはわからない、つまり、どの命令をフェッチすれば良いのかわからないわけである。

しかし、実際には、CPU は分岐条件となる計算結果が得られるまで命令フェッチを待ち続けるわけではなく、実際には分岐の方向を予測 (Branch Prediction) して投機的に実行 (Speculative Execution) し、その予測が合っていることがわかった場合は投機的な実行の結果を利用し、そうでない場合はその結果を捨てて正しい分岐の方の実行を行う。

つまり、この分岐予測がより正確にできればできるほど、CPU のパフォーマンスをよくなる。


# `likely()` / `unlikely()`

`likely()` / `unlikely()` は、コンパイラに分岐予測のためのヒントを与えるためのマクロである。
その名の通り、`likely()` はその条件が満たされることが期待される場合に、`unlikely()` はその条件が満たされないことが期待される場合に用いる。

以下のように、`__builtin_expect()` を使っている (詳細は、後述) 。

https://elixir.bootlin.com/linux/v6.9.8/source/include/linux/compiler.h#L76
```c
# define likely(x)	__builtin_expect(!!(x), 1)
# define unlikely(x)	__builtin_expect(!!(x), 0)
```

`!!(x)` を使う理由は、与えられた値が 0 または 1 になるようにするためである。
具体的には、0 は 0 のままになり、0 以外の値は 1 になる。

以下のように、実際に実行結果を見てみる方が早いだろう。

```c
#include <stdio.h>

int main() {
	printf("!(0)   => %d, ", !(0));
	printf("!!(0)  => %d\n", !!(0));

	printf("!(1)   => %d, ", !(1));
	printf("!!(1)  => %d\n", !!(1));

	printf("!(2)   => %d, ", !(2));
	printf("!!(2)  => %d\n", !!(2));

	printf("!(-1)  => %d, ", !(-1));
	printf("!!(-1) => %d\n", !!(-1));

	return 0;
}
```
```
$ gcc a.c
$ ./a.out
!(0)   => 1, !!(0)  => 0
!(1)   => 0, !!(1)  => 1
!(2)   => 0, !!(2)  => 1
!(-1)  => 0, !!(-1) => 1
```


# `__builtin_expect()`

以下の GCC のドキュメントにある通り、`long __builtin_expect (long exp, long c)` は、90% 程度の確率で `exp == c` であることを期待していると、分岐予測のための情報としてコンパイラに伝えるものである。

https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#index-_005f_005fbuiltin_005fexpect

> Built-in Function: `long __builtin_expect (long exp, long c)`
>
> You may use `__builtin_expect` to provide the compiler with branch prediction information. In general, you should prefer to use actual profile feedback for this (`-fprofile-arcs`), as programmers are notoriously bad at predicting how their programs actually perform. However, there are applications in which this data is hard to collect.
>
> The return value is the value of `exp`, which should be an integral expression. The semantics of the built-in are that it is expected that `exp == c`. For example:
> 
> ```
> if (__builtin_expect (x, 0))
>   foo ();
> ```
> 
> indicates that we do not expect to call foo, since we expect x to be zero. Since you are limited to integral expressions for exp, you should use constructions such as
> 
> ```
> if (__builtin_expect (ptr != NULL, 1))
>   foo (*ptr);
> ```
> 
> when testing pointer or floating-point values.
> 
> For the purposes of branch prediction optimizations, the probability that a `__builtin_expect` expression is `true` is controlled by GCC’s `builtin-expect-probability` parameter, which defaults to 90%.
> 
> You can also use `__builtin_expect_with_probability` to explicitly assign a probability value to individual expressions. If the built-in is used in a loop construct, the provided probability will influence the expected number of iterations made by loop optimizations.


# Linux kernel 内での利用例

以下は、システムコールを呼び出す時に使われる関数で、与えられたシステムコール番号が最大値 (`__NR_syscalls`) よりも小さいことが期待される。

https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L50

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
		regs->ax = x64_sys_call(regs, unr);
		return true;
	}
	return false;
}
```
