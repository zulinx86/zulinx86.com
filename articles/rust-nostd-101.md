---
title: 【Rust】"Hello world!" without Standard Library
emoji: "🦀"
type: "idea"
topics: ["rust"]
published: true
---

# Without Standard Library

Let's start with a normal "Hello, world!" program as follows:

```rs
// main.rs

fn main() {
    println!("Hello, world!");
}
```

https://doc.rust-lang.org/reference/names/preludes.html#the-no_std-attribute

> By default, the standard library is automatically included in the crate root module. The `std` crate is added to the root, along with an implicit `marco_use` attribute pulling in all marcos exported from `std` into the `marco_use` prelude.

> The`no_std` attribute may be applied at the crate level to prevent the `std` crate from being automatically added into scope.

https://docs.rust-embedded.org/book/intro/no-std.html

> `#![no-std]` is a crate-level attribute that indicates that the crate will link the core-crate instead of the std-crate. The libcore crate in turn is a platform-agnostic subset of the std crate whcih makes no assumptions about the system the program will run on.

> no_std and libcore code can be used for any kind of bootstrapping (stage 0) code like bootloaders, firmware or kernels.

So let's add `#![no_std]` and try to compile it!

```rs
// main.rs

#![no_std] // don't link the Rust standard library

fn main() {
    println!("Hello, world!");
}
```
```
$ cargo build
   Compiling baremetal_rust v0.1.0 (/workplace/baremetal_rust)
error: cannot find macro `println` in this scope
 --> src/main.rs:6:5
  |
6 |     println!("Hello, world!");
  |     ^^^^^^^

error: `#[panic_handler]` function required, but not found

error: unwinding panics are not supported without std
  |
  = help: using nightly cargo, use -Zbuild-std with panic="abort" to avoid unwinding
  = note: since the core library is usually precompiled with panic="unwind", rebuilding your crate with panic="abort" may not be enough to fix the problem

error: could not compile `baremetal_rust` (bin "baremetal_rust") due to 3 previous errors
```

The compile fails with three errors:
- `println!()` is not available.
- `#[panic_handler]` function is required.
- unwinding panics are not supported without standard library.

Let's go through one by one.

# `#[panic_handler]` function

https://os.phil-opp.com/freestanding-rust-binary/#panic-implementation

> ### Panic Implementation
>
> The `panic_handler` attribute defines the function that the compiler should invoke when a panic occurs. The standard library provides its own panic handler function, but in a `no_std` environment we need to define it ourselves:
>
> ```rs
> // in main.rs
>
> use core::panic::PanicInfo
>
> /// This function is called on panic.
> #[panic_handler]
> fn panic(_info: &PanicInfo) -> ! {
>     loop {}
> }
> ```
>
> The `PanicInfo` parameter contains the file and line where the panic happened and the optional panic message. The function should never return, so it is marked as a diverging function by returning the "never" type `!`.

So let's add the panic handler as it is!

# Disabling Stack Unwinding

https://os.phil-opp.com/freestanding-rust-binary/#the-eh-personality-language-item

> ### The `eh_personality` Language Item
>
> Language items are special functions an types that are required internally by the compiler. For example, the `Copy` trait is a language item that tells the compiler which types have copy semantics. When we look at the implementation, we see it has the special `#[lang = "copy"]` attribute that defines it as a language item.
>
> While providing custom implementations of language items is possible, it should only be done as a last resort. The reason is that language items are highly unstable implementation details and not even type checked (so the compiler doesn't even check if a function has the right argument types). Fortunately, there is a more stable way to fix the above language item error.
>
> The `eh_personality` language item marks a function that is used for implementing stack unwiding. By default, Rust uses unwinding to run the destructors of all live stack variables in case of a panic. This ensures that all used memory is freed and allows the parent thread to catch the panic and continue execution. Unwinding, however, is a complicated process and requries some OS-specific libraries (e.g. libunwind on Linux or structured exception handling on Windows), so we don't want to use it for our operating system.
>
> #### Disabling Unwinding
>
> There are other use cases as well for which unwinding is undesirable, so Rust provides an option to abort on panic instead. This disables the generation of unwinding symbol informationand thus considerably reduces binary size. There are multiple places where we can disable unwinding. The easiest way is to add the following lines to our `Cargo.toml`:
>
> ```toml
> [profile.dev]
> panic = "abort"
>
> [profile.release]
> panic = "abort"
> ```
>
> This sets the panic strategy to `abort` for both the `dev` profile (used for `cargo build`) and the `release` profile (used for `cargo build --release`). Now the `eh_personality` language item should no longer be required.

So let's set `panic = "abort"` in `Cargo.toml`.

# No Main Function

Now we should have fixed two of three errors. Let's compile it without `println!()`!

```rs
// main.rs

#![no_std] // don't link the Rust standard library

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

fn main() {}
```
```toml
# Cargo.toml

[package]
name = "baremetal_rust"
version = "0.1.0"
edition = "2021"

# the profile used for `cargo build`
[profile.dev]
panic = "abort" # disable stack unwinding on panic

# the profile used for `cargo build --release`
[profile.release]
panic = "abort" # disable stack unwinding on panic
```

```
$ cargo build
   Compiling baremetal_rust v0.1.0 (/workplace/baremetal_rust)
error: using `fn main` requires the standard library
  |
  = help: use `#![no_main]` to bypass the Rust generated entrypoint and declare a platform specific entrypoint yourself, usually with `#[no_mangle]`

error: could not compile `baremetal_rust` (bin "baremetal_rust") due to 1 previous error
```

It complains `main()` requires the standard library and we need to add `#![no_main]` and declare a platform specific entrypoint with `#[no_mangle]`.

Let's remove `main()` and add `#![no_main]` and compile it any way!

```rs
// main.rs

#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

```
$ cargo build
   Compiling baremetal_rust v0.1.0 (/workplace/baremetal_rust)
error: linking with `cc` failed: exit status: 1
  |
...
  = note: /usr/lib/gcc/x86_64-redhat-linux/7/../../../../lib64/Scrt1.o: In function `_start':
          (.text+0x12): undefined reference to `__libc_csu_fini'
          (.text+0x19): undefined reference to `__libc_csu_init'
          (.text+0x20): undefined reference to `main'
          (.text+0x26): undefined reference to `__libc_start_main'
          collect2: error: ld returned 1 exit status

  = note: some `extern` functions couldn't be found; some native libraries may need to be installed or have their path specified
...
```

It still fails with a linker error.

https://os.phil-opp.com/freestanding-rust-binary/#the-start-attribute

> ### The `start` attribute
>
> One might think that the `main` function is the first function called when you run a program. However, most languages have a runtime system, which is responsible for things such as garbage collection (e.g. in Java) or software threads (e.g. goroutines in Go). This runtime needs to be called before `main`, since it needs to initialize itself.
>
> In a typical Rust binary that links the standard library, execution starts in a C runtime library called `crt0` ("C runtime zero"), which sets up the environment for a C application. This includes creating a stack and placing the arguments in the right registers. The C runtime then invokes the entry point of the Rust runtime, which is marked by the `start` language item. Rust only has a very minimal runtime, which takes care of some small things such as setting up stack overflow guards or printing a backtrace on panic. The runtime then finally calls the `main` function.
>
> Our freestanding executable does not have access to the Rust runtime and `crt0`, so we need to define our own entry point. Implementing the `start` language item wouldn't help, since it would still require `crt0`. Instead, we need to overwrite the `crt0` entry point directly.
>
> #### Overwriting the Entry Point
>
> To tell the Rust compiler that we don't want to use the normal entry point chain, we add the `#![no_main]` attribute.
>
> ```rs
> #![no_std]
> #![no_main]
>
> use core::panic::PanicInfo;
>
> /// This function is called on panic.
> #[panic_handler]
> fn panic(_info: &PanicInfo) -> ! {
>     loop {}
> }
> ```
>
> You might notice that we removed the `main` function. The reason is that a `main` doesn't make sense without an underlying runtime that calls it. Instead, we are now overwriting the operating system entry point with our own `_start` function:
>
> ```rs
> #[no_mangle]
> pub extern "C" fn _start() -> ! {
>     loop {}
> }
> ```
>
> By using the `#![no_mangle]` attribute, we disable name mangling to ensure that the Rest compiler really outputs a function with the name `_start`. Without the attribute, the compiler would generate some cryptic `_ZN3blog_os4_start7hb173fedf945531caE` symbol to give every function a unique name. The attribute is required because we need to tell the name of the entry point function to the linker in the next step.
>
> We also have to mark the function as `extern "C"` to tell the compiler that it should use the C calling convention for this function (instead of the unspecified Rust calling convention). The reason for naming the function `_start` is that this is the default entry point name for most systems.

So let's add `_start()`.

```rs
// main.rs

#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo;

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

```

It's still complaining the linker error, but now it says there are multiple definition of `_start`.

```
$ cargo build
   Compiling baremetal_rust v0.1.0 (/workplace/baremetal_rust)
error: linking with `cc` failed: exit status: 1
  |
...
  = note: /workplace/baremetal_rust/target/debug/deps/baremetal_rust-857642be75594060.572hqs9muqaokms2.rcgu.o: In function `_start':
          /workplace/baremetal_rust/src/main.rs:12: multiple definition of `_start'
          /usr/lib/gcc/x86_64-redhat-linux/7/../../../../lib64/Scrt1.o:(.text+0x0): first defined here
          /usr/lib/gcc/x86_64-redhat-linux/7/../../../../lib64/Scrt1.o: In function `_start':
          (.text+0x12): undefined reference to `__libc_csu_fini'
          (.text+0x19): undefined reference to `__libc_csu_init'
          (.text+0x20): undefined reference to `main'
          (.text+0x26): undefined reference to `__libc_start_main'
          collect2: error: ld returned 1 exit status
```

https://os.phil-opp.com/freestanding-rust-binary/#linker-errors

> ### Linker Errors
>
> The linker is a program that combines the generated code into an executable. Since the executable format differs from Linux, Windows, and macOS, each system has its own linker that throws a different error. The fundamental cause of the errors is the same: the default configuration of the linker assumes that our program depends on the C runtime, which it does not.
>
> To solve the errors, we need to tell the linker that it should not include the C runtime. We can do this either by passing a certain set of arguments to the linker or by building for a bare metal target.
>
> #### Building for a Bare Metal Target
>
> By default Rust tries to build an executable that is able to run in your current system environment. For example, if you're using Windows on `x86_64`, Rust tries to build an `.exe` Windows executable that uses `x86_64` instructions. The environment is called your "host" system.
>
> To describe different environments, Rust uses a string called target triple. You can see the target triple for your host system by running `rustc --version --verbose`:
>
> ```
> rustc 1.35.0-nightly (474e7a648 2019-04-07)
> binary: rustc
> commit-hash: 474e7a6486758ea6fc761893b1a49cd9076fb0ab
> commit-date: 2019-04-07
> host: x86_64-unknown-linux-gnu
> release: 1.35.0-nightly
> LLVM version: 8.0
> ```
>
> The above output is from a `x86_64` Linux system. We see that the `host` triple is `x86_64-unknown-linux-gnu`, which includes the CPU architecture (`x86_64`), the vendor (`unknown`), the operating system (`linux`), and the ABI (`gnu`).
>
> By compiling for our host triple, the Rust compiler and the linker assume that there is an underlying operating system such as Linux or Windows that uses the C runtime by default, which causes the linker errors. So, to avoid the linkner errors, we can compile for a different environment with no underlying operating system.

> By passing a `--target` argument we cross compile our executable for a bare metal target system. Since the target system has no operating system, the linker does not try to link the C runtime and our build succeeds without any linker errors.

To tell the linker that it should not include the C runtime, let's specify `x86-64-unknown-none` as target.

```
$ cargo build --target x86_64-unknown-none
   Compiling baremetal_rust v0.1.0 (/workplace/itazur/test/nostd_rust)
    Finished dev [unoptimized + debuginfo] target(s) in 0.05s
```

It can also be specified in `.cargo/config.toml`

```toml
# .cargo/config.toml

[build]
target = ["x86_64-unknown-none"]
```
```
$ cargo build
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
```

# Printing "Hello, world!"

To print "Hello, world!", we'll call `write()` system call and use assembly for system calls.

You can write assembly code in Rust using `asm!` macro.

https://doc.rust-lang.org/reference/inline-assembly.html

> Support for inline assembly is provided via the `asm!` and `global_asm!` marcos.

> With the `asm!` macro, the assembly code is emitted in a function scope and integrated into the compiler-generated assembly code of a function. This assembly code must obey strict rules to avoid undefined behavior. Note that in some cases the compiler may choose to emit the assembly code as a separate function and generate a call to it.

> An `asm!` invocation may have one or more template string arguments; an `asm!` with multiple template string arguments is treated as if all the strings were concatenated with a `\n` between them. The expected usage is for each template string argument to correspond to a line of assembly code. All template string arguments must appear before any other arguments.

> Several types of operands are suppored:
> - `in(<reg>) <expr>`
>     - `<reg>` can refer to a register class or an explicit register. The allocated register name is substituted into the asm template string.
>     - The allocated register will contain the value of `<expr>` at the start of the asm code.
>     - The allocated register must contain the same value at the end of the asm code (except if a `lateout` is allocated to the same register).
> - `out(<reg>) <expr>`
>     - `<reg>` can refer to a register class or an explicit register. The allocated register name is substituted into the asm template string.
>     - The allocated register will contain an undefined value at the start of the asm code.
>     - `<expr>` must be a (possibly uninitialized) place expression, to which the contents of the allocated register are written at the end of the asm code.
>     - An underscore (`_`) may be specified instead of an expression, which will cause the contents of the register to be discarded at the end of the asm code.
> - `lateout(<reg>) <expr>`
>     - Identical to `out` except that the register allocator can reuse a register allocated to an `in`.
>     - You should only write to the register after all inputs are read, otherwise you may clobber an input.

Next, let's see how to call system calls in assembly.

https://man7.org/linux/man-pages/man2/syscall.2.html

> ```
>        Arch/ABI    Instruction           System  Ret  Ret  Error    Notes
>                                          call #  val  val2
>        ───────────────────────────────────────────────────────────────────
> ...
>        x86-64      syscall               rax     rax  rdx  -        5
> ```

> ```
>        Arch/ABI      arg1  arg2  arg3  arg4  arg5  arg6  arg7  Notes
>        ──────────────────────────────────────────────────────────────
> ...
>        x86-64        rdi   rsi   rdx   r10   r8    r9    -
> ```

https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl

> 1	common	write			sys_write

https://man7.org/linux/man-pages/man2/write.2.html

> ```c
>        #include <unistd.h>
>
>        ssize_t write(int fd, const void buf[.count], size_t count);
> ```

https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf

> 1. User-level applications use as integer registers for passing the sequence `%rdi`, `%rsi`, `%rdx`, `%rcx`, `%r8` and `%r9`. The kernel interface uses `%rdi`, `%rsi`, `%rdx`, `%r10`, `%r8` and `%r9`.
> 2. A system-call is done via `syscall` instruction. The kernel destroys registers `%rcx` and `%r11`.
> 3. The number of the syscall has to be passed in register `%rax`.
> 4. System-calls are limited to six arguments, no argument is passed directly on the stack.
> 5. Returning from the `syscall`, register `%rax` contains the result of the system-call. A value in the range between -4095 and -1 indicates an error, it is `-errno`.
> 6. Only values of class INTEGER or class MEMORY are passed to the kernel.

Let's call `write()` system call!

```rs
// main.rs

#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::arch::asm;
use core::panic::PanicInfo;

// https://man7.org/linux/man-pages/man2/write.2.html
// ```c
// ssize_t write(int fd, const void buf[.count], size_t count);
// ```
fn sys_write(fd: i32, buf: *const u8, count: usize) -> usize {
    unsafe {
        let ret: usize;

        asm!(
            "syscall",
            in("rax") 1,
            in("rdi") fd,
            in("rsi") buf,
            in("rdx") count,
            lateout("rax") ret,
            out("rcx") _,
            out("r11") _,
        );

        ret
    }
}

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default

    let msg = b"Hello, world!\n";
    let ret = sys_write(1, msg.as_ptr(), msg.len());
    assert!(ret == msg.len());

    loop {}
}

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```
```
$ cargo run
   Compiling baremetal_rust v0.1.0 (/workplace/baremetal_rust)
    Finished dev [unoptimized + debuginfo] target(s) in 0.09s
     Running `target/x86_64-unknown-none/debug/baremetal_rust`
Hello, world!

```

# Exit with 0 on Success

Now it loops infinitely. So let's make it exit with 0 on success.

https://man7.org/linux/man-pages/man3/exit.3.html

> ```c
>        #include <stdlib.h>
>
>        [[noreturn]] void exit(int status);
> ```

https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl

> 60	common	exit			sys_exit

https://doc.rust-lang.org/reference/inline-assembly.html

> - `noreturn`: The `asm!` block never returns, and its return type is defined as `!` (never). Behavior is undefined if execution falls through past the end of the asm code. A `noreturn` asm block behaves just like a function which doesn't return; notably, local variables in scope are not dropped before it is invoked.

```rs
// main.rs

#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::arch::asm;
use core::panic::PanicInfo;

// https://man7.org/linux/man-pages/man2/write.2.html
// ```c
// ssize_t write(int fd, const void buf[.count], size_t count);
// ```
fn sys_write(fd: i32, buf: *const u8, count: usize) -> usize {
    unsafe {
        let ret: usize;

        asm!(
            "syscall",
            in("rax") 1,
            in("rdi") fd,
            in("rsi") buf,
            in("rdx") count,
            lateout("rax") ret,
            out("rcx") _,
            out("r11") _,
        );

        ret
    }
}

// https://man7.org/linux/man-pages/man3/exit.3.html
// ```c
// [[noreturn]] void exit(int status);
// ```
fn sys_exit(status: i32) -> ! {
    unsafe {
        asm!(
            "syscall",
            in("rax") 60,
            in("rdi") status,
            options(noreturn)
        );
    }
}

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default

    let msg = b"Hello, world!\n";
    let ret = sys_write(1, msg.as_ptr(), msg.len());
    if ret != msg.len() {
        sys_exit(1);
    }

    sys_exit(0);
}

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```
```
$ cargo run
   Compiling baremetal_rust v0.1.0 (/workplace/baremetal_rust)
    Finished dev [unoptimized + debuginfo] target(s) in 0.09s
     Running `target/x86_64-unknown-none/debug/baremetal_rust`
Hello, world!

$ echo $?
0
```

# Final Code

```toml
# .cargo/config.toml

[build]
target = ["x86_64-unknown-none"]
```
```toml
# Cargo.toml

[package]
name = "baremetal_rust"
version = "0.1.0"
edition = "2021"

[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```
```rs
// main.rs

#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::arch::asm;
use core::panic::PanicInfo;

// https://man7.org/linux/man-pages/man2/write.2.html
// ```c
// ssize_t write(int fd, const void buf[.count], size_t count);
// ```
fn sys_write(fd: i32, buf: *const u8, count: usize) -> usize {
    unsafe {
        let ret: usize;

        asm!(
            "syscall",
            in("rax") 1,
            in("rdi") fd,
            in("rsi") buf,
            in("rdx") count,
            lateout("rax") ret,
            out("rcx") _,
            out("r11") _,
        );

        ret
    }
}

// https://man7.org/linux/man-pages/man3/exit.3.html
// ```c
// [[noreturn]] void exit(int status);
// ```
fn sys_exit(status: i32) -> ! {
    unsafe {
        asm!(
            "syscall",
            in("rax") 60,
            in("rdi") status,
            options(noreturn)
        );
    }
}

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default

    let msg = b"Hello, world!\n";
    let ret = sys_write(1, msg.as_ptr(), msg.len());
    if ret != msg.len() {
        sys_exit(1);
    }

    sys_exit(0);
}

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

```

# References

- [A Freestanding Rust Binary | Writing an OS in Rust](https://os.phil-opp.com/freestanding-rust-binary/)
- [Learning Rust: Nothing. Imagine you woke up with Rust in one… | by Adrian Macal | Mar, 2024 | Level Up Coding](https://levelup.gitconnected.com/learning-rust-nothing-19073fa0e69b)
- [no_std - The Embedded Rust Book](https://docs.rust-embedded.org/book/intro/no-std.html)
- [Preludes - The Rust Reference](https://doc.rust-lang.org/reference/names/preludes.html#the-no_std-attribute)
- [Crates and source files - The Rust Reference](https://doc.rust-lang.org/reference/crates-and-source-files.html#the-no_main-attribute)
- [Inline assembly - The Rust Reference](https://doc.rust-lang.org/reference/inline-assembly.html)
