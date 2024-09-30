---
title: 【Python】threading
emoji: "🐍"
type: "tech"
topics: ["python", "threading"]
published: true
---

Python でスレッドの使い方をざっくりと把握していきます。



# 使ってみる

## １スレッド

まずはともあれスレッドを作成、実行していきます。

```py
import time
import datetime
import threading

def func():
    print(
        "Thread:",
        threading.current_thread().name,
        "starting at",
        datetime.datetime.now()
    )

    # sleep 3 seconds
    time.sleep(3)

    print(
        "Thread:",
        threading.current_thread().name,
        "ending at",
        datetime.datetime.now()
    )

if __name__ == '__main__':
    print(
        "Thread:",
        threading.current_thread().name,
        "starting at",
        datetime.datetime.now()
    )

    # create a thread
    t = threading.Thread(target=func)

    # start the thread
    t.start()

    # wait for the thread
    t.join()

    print(
        "Thread:",
        threading.current_thread().name,
        "ending at",
        datetime.datetime.now()
    )
```
```
$ python3 main.py
Thread: MainThread starting at 2024-09-23 14:31:02.433882
Thread: Thread-1 (func) starting at 2024-09-23 14:31:02.434125
Thread: Thread-1 (func) ending at 2024-09-23 14:31:05.437251
Thread: MainThread ending at 2024-09-23 14:31:05.437520
```

`MainTheead` が、作成された `Thread-1` を待ってから終了してます。


## ２スレッド

スレッドを 2 つにしてみましょう。

```py
import time
import datetime
import threading

def func():
    print(
        "Thread:",
        threading.current_thread().name,
        "starting at",
        datetime.datetime.now()
    )

    # sleep 3 seconds
    time.sleep(3)

    print(
        "Thread:",
        threading.current_thread().name,
        "ending at",
        datetime.datetime.now()
    )

if __name__ == '__main__':
    print(
        "Thread:",
        threading.current_thread().name,
        "starting at",
        datetime.datetime.now()
    )

    # create threads
    t1 = threading.Thread(target=func)
    t2 = threading.Thread(target=func)

    # start the threads
    t1.start()
    t2.start()

    # wait for the threads
    t1.join()
    t2.join()

    print(
        "Thread:",
        threading.current_thread().name,
        "ending at",
        datetime.datetime.now()
    )
```
```
$ python3 main.py
Thread: MainThread starting at 2024-09-23 14:34:00.347718
Thread: Thread-1 (func) starting at 2024-09-23 14:34:00.347962
Thread: Thread-2 (func) starting at 2024-09-23 14:34:00.348298
Thread: Thread-1 (func) ending at 2024-09-23 14:34:03.351087
Thread: Thread-2 (func) ending at 2024-09-23 14:34:03.351431
Thread: MainThread ending at 2024-09-23 14:34:03.351661
```

ちゃんと２つのスレッドが並行で実行されています。


## `join()` しなかったら？

```
$ python3 main.py
Thread:  MainThread starting at 2024-09-23 14:38:49.934341
Thread:  Thread-1 (func) starting at 2024-09-23 14:38:49.934598
Thread:  Thread-2 (func) starting at 2024-09-23 14:38:49.934924
Thread:  MainThread ending at 2024-09-23 14:38:49.935032
Thread:  Thread-1 (func) ending at 2024-09-23 14:38:52.937704
Thread:  Thread-2 (func) ending at 2024-09-23 14:38:52.937926
```

先に `MainThread` が終了し、その後で `Thread-1`、`Thread-2` が処理完了時に終了しています。



# 公式ドキュメント

使い方をちょっと理解したら、ちゃんと公式ドキュメントをさらっとみておきましょう。

https://docs.python.org/3/library/threading.html

> CPython implementation detail: In CPython, due to the Global Interpreter Lock, only one thread can execute Python code at once (even though certain performance-oriented libraries might overcome this limitation). If you want your application to make better use of the computational resources of multi-core machines, you are advised to use multiprocessing or concurrent.futures.ProcessPoolExecutor. However, threading is still an appropriate model if you want to run multiple I/O-bound tasks simultaneously.

CPython では、一度に実行されるスレッドは１つのみであるとのこと。仮にマルチコアの CPU を使っていたとしても、１つのスレッドしか一つのタイミングでは実行してくれない。より効率的にマルチコアを使いたければ、multiprocessing または concurrent.features.ProcessPoolExecutor を使いなさいとのこと。とはいえ、I/O が主な処理な場合は、I/O 待ちの時間がメインであり、スレッドを並列で実行できなくても大丈夫なので、threading でも十分ですよと。


## `threading.Thread`

### スレッドの実行の仕方

https://docs.python.org/3/library/threading.html#thread-objects

> There are two ways to specify the activity: by passing a callable object to the constructor, or by overriding the run() method in a subclass.

実行する内容を指定する方法は２つあります。
1. コンストラクタに呼び出し可能なオブジェクトを渡す。
2. サブクラスで `run()` を上書きする。

> start()
> Start the thread’s activity.
> 
> It must be called at most once per thread object. It arranges for the object’s run() method to be invoked in a separate thread of control.
> 
> This method will raise a RuntimeError if called more than once on the same thread object.

`start()` は、オブジェクトの `run()` が別の制御スレッドで呼び出さられるようにしてくれるものです。

> run()
>
> Method representing the thread’s activity.
>
> You may override this method in a subclass. The standard run() method invokes the callable object passed to the object’s constructor as the target argument, if any, with positional and keyword arguments taken from the args and kwargs arguments, respectively.
> 
> Using list or tuple as the args argument which passed to the Thread could achieve the same effect.

`run()` は、サブクラス側で上書きすることができます。

デフォルトの実装では、`target` で渡された呼び出し可能なオブジェクトに `args` と `kwargs` を渡して呼び出します。

> Once a thread object is created, its activity must be started by calling the thread’s start() method. This invokes the run() method in a separate thread of control.

スレッドオブジェクトが作られると、`start()` で実行を開始し、内部的に `run()` を異なるスレッドで実行されます。

サブクラスを使った場合は、以下のようになります。

```py
import time
import datetime
import threading

class t(threading.Thread):
    def run(self):
        print(
            "Thread:",
            self.name,
            "starting at",
            datetime.datetime.now()
        )

        # sleep 3 seconds
        time.sleep(3)

        print(
            "Thread:",
            self.name,
            "ending at",
            datetime.datetime.now()
        )

if __name__ == '__main__':
    print(
        "Thread:",
        threading.current_thread().name,
        "starting at",
        datetime.datetime.now()
    )

    # create threads
    t1 = t()
    t2 = t()

    # start the threads
    t1.start()
    t2.start()

    # wait for the threads
    t1.join()
    t2.join()

    print(
        "Thread:",
        threading.current_thread().name,
        "ending at",
        datetime.datetime.now()
    )
```
```
$ python3 main.py
Thread: MainThread starting at 2024-09-23 15:04:12.154180
Thread: Thread-1 starting at 2024-09-23 15:04:12.154412
Thread: Thread-2 starting at 2024-09-23 15:04:12.154749
Thread: Thread-1 ending at 2024-09-23 15:04:15.157531
Thread: Thread-2 ending at 2024-09-23 15:04:15.157857
Thread: MainThread ending at 2024-09-23 15:04:15.158100
```

> Once the thread’s activity is started, the thread is considered ‘alive’. It stops being alive when its run() method terminates – either normally, or by raising an unhandled exception. The is_alive() method tests whether the thread is alive.

`start()` が実行されると 'alive' の状態になり、`run()` が終了する or 例外が発生すると 'alive' でなくなります。

### `join()`

> Other threads can call a thread’s join() method. This blocks the calling thread until the thread whose join() method is called is terminated.

別のスレッドが、あるスレッドに対して `join()` を呼び出すことができ、呼び出した側のスレッドは、呼び出された側のスレッドが終わるまでブロックされます。

### デーモンスレッド

> A thread can be flagged as a “daemon thread”. The significance of this flag is that the entire Python program exits when only daemon threads are left. The initial value is inherited from the creating thread. The flag can be set through the daemon property or the daemon constructor argument.

スレッドを、デーモンスレッドすることができます。これは、デーモンスレッド以外の実行が終了した場合に、Python プログラム全体が終了します。

先ほどの例では、デーモン化しなかったために、MainThread が先に終了しても Thread-1 / Thread-2 の終了を待ってしまったですね。

では、デーモン化した場合を見てみましょう。

```py
import time
import datetime
import threading

def func():
    print(
        "Thread:",
        threading.current_thread().name,
        "starting at",
        datetime.datetime.now()
    )

    # sleep 3 seconds
    time.sleep(3)

    print(
        "Thread:",
        threading.current_thread().name,
        "ending at",
        datetime.datetime.now()
    )

if __name__ == '__main__':
    print(
        "Thread:",
        threading.current_thread().name,
        "starting at",
        datetime.datetime.now()
    )

    # create threads
    t1 = threading.Thread(target=func, daemon=True)
    t2 = threading.Thread(target=func, daemon=True)

    # start the threads
    t1.start()
    t2.start()

    # don't wait for the threads
    #t1.join()
    #t2.join()

    print(
        "Thread:",
        threading.current_thread().name,
        "ending at",
        datetime.datetime.now()
    )
```
```
$ python3 main.py 
Thread: MainThread starting at 2024-09-23 15:12:22.520531
Thread: Thread-1 (func) starting at 2024-09-23 15:12:22.520768
Thread: Thread-2 (func) starting at 2024-09-23 15:12:22.521110
Thread: MainThread ending at 2024-09-23 15:12:22.521134
```

MainThread が終了した時点で、残りのスレッド (Thread-1 / Thread-2) はデーモンスレッドなので、プログラム全体が終了してくれました。
