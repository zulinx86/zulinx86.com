---
title: 【Python】型ヒント
emoji: "🐍"
type: "tech"
topics: ["python"]
published: true
---


Python は動的型付け言語として有名であり、実行時に変数の型が決定される。
これによって、非常に楽にコーディングができる一方で、コンパイルしないために実行時になって初めてエラーが検出されるといった欠点もある。



# 関数アノテーション

Python 3.0 から PEP 3107 で関数アノテーションが導入された。

https://peps.python.org/pep-3107/

> 1. Function annotations, both for parameters and return values, are completely optional.

関数アノテーションは、引数や返り値の両方に対して、オプションでつけることができる。

> 2. Function annotations are nothing more than a way of associating arbitrary Python expressions with various parts of a function at compile-time.

関数アノテーションは、任意の Python の式 (expression) を関数に紐付けることができる。

> By itself, Python does not attach any particular meaning or significance to annotations.

Python はアノテーションに対して、何ら特別な意味や意義を持たせていない。

> The only way that annotations take on meaning is when they are interpreted by third-party libraries. These annotation consumers can do anything they want with a function’s annotations.

アノテーションが意味を持つようになるのは、サードパーティ製のライブラリによって解釈されたときだけである。そのライブラリは好きなようにアノテーションを利用することができる。

> 3. Following from point 2, this PEP makes no attempt to introduce any kind of standard semantics, even for the built-in types. This work will be left to third-party libraries.

PEP は、ビルトインの型も含めて、何らセマンティクスを導入しない (= 意味を持たせない) 。解釈は完全にサードパーティ製のライブラリ次第である。


要するに、サードパーティ製のライブラリを導入しない限りは、アノテーションで型を指定したとしても、その型と違う型の変数を渡すことができてしまう。


## シンタックス

### 引数

引数に対するアノテーションのシンタックスは以下の通り。

```py
identifier [:expression] [= expression]
```

なお、`= expression` の部分は、デフォルト引数を提供するものである。

例えば、以下のように使える。

```py
def func(a: expression, b: expression = 5):
    ...
```

### 返り値

シンタックスとしては、`->` の後ろに任意の式を与えられる。

```py
fn func() -> expression:
    ...
```


## 関数アノテーションへのアクセス

`__annotations__` アトリビュートを通して、関数アノテーションにアクセスすることが可能である。

```py
>>> name: str = "hello"
>>> __annotations__
{'name': <class 'str'>}
```



# 型ヒント

Python 3.5 から PEP 484 で型ヒントが導入された。

https://peps.python.org/pep-0484/

> PEP 3107 introduced syntax for function annotations, but the semantics were deliberately left undefined.

上述の通り、PEP 3107 で型アノテーションのためのシンタックスが導入されたが、そのセマンティクスについては意図的に未定義としておいた。

> There has now been enough 3rd party usage for static type analysis that the community would benefit from a standard vocabulary and baseline tools within the standard library.

時が経ち、静的型分析のためのサードパーティ製ツールが使用されるようになり、標準ライブラリ内に標準的なツールを導入する時がきた。

> This PEP introduces a provisional module to provide these standard definitions and tools, along with some conventions for situations where annotations are not available.

この PEP で、その標準的な定義とツールを導入する暫定的なモジュールを導入する。

> For example, here is a simple function whose argument and return type are declared in the annotations:
> 
> ```py
> def greeting(name: str) -> str:
>     return 'Hello ' + name
> ```
> 
> While these annotations are available at runtime through the usual __annotations__ attribute, no type checking happens at runtime. Instead, the proposal assumes the existence of a separate off-line type checker which users can run over their source code voluntarily. Essentially, such a type checker acts as a very powerful linter. (While it would of course be possible for individual users to employ a similar checker at run time for Design By Contract enforcement or JIT optimization, those tools are not yet as mature.)

上述の通り、`__annotations__` を通して、アノテーションにアクセスすることができるが、実行時には一切の型チェックは行わない。

その代わりに、ユーザーが実行時外の好きなタイミングで実行できる型チェッカーを前提とする。この型チェッカーは、強力なリンターとして利用することができる。

> The proposal is strongly inspired by mypy.

この PEP は、mypy に強くインスパイアされている。

> The type system supports unions, generic types, and a special type named Any which is consistent with (i.e. assignable to and from) all types. This latter feature is taken from the idea of gradual typing. Gradual typing and the full type system are explained in PEP 483.

この型システムでは、ユニオン型、ジェネリック型、任意の型を受け取れる特別な型などをサポートする。

型付けシステムについては、PEP 483 で定義されている。

https://peps.python.org/pep-0483/


型ヒントに関するチートシートが、以下のリンクから利用可能である。

https://mypy.readthedocs.io/en/stable/cheat_sheet_py3.html


## typing

https://docs.python.org/3/library/typing.html

typing モジュールは、より複雑な型ヒントを利用できるようにするためのものである。
例えば、`List`、`Set`、`Dict`、`Tuple`、`Union`、`Optional`、`Callable` などである。

なお、Python 3.9 以降では、頭文字が小文字の `list`、`set`、`dict`、`tuple` がサポートされている。
