---
layout: post
title: Debug
---



## Debug with GDB
1. Initialize .config.
  ```
  $ make x86_64_defconfig
  ```
1. Customize .config.
  ```
  $ make menuconfig
  ```
	1. Enable `CONFIG_DEBUG_KERNEL`.
    ```
    Symbol: DEBUG_KERNEL [=y]
    Type  : bool
    Defined at lib/Kconfig.debug:588
      Prompt: Kernel debugging
      Location:
        Main menu
    (1)   -> Kernel hacking
    ```
	1. Enable `CONFIG_DEBUG_INFO`.
    ```
    Symbol: DEBUG_INFO [=y]
    Type  : bool
    Defined at lib/Kconfig.debug:213
      Prompt: Compile the kernel with debug info
      Depends on: DEBUG_KERNEL [=y] && !COMPILE_TEST [=n]
      Location:
        Main menu
          -> Kernel hacking
    (1)     -> Compile-time checks and compiler options
    ```
1. Build.
```
$ make -j$(nproc)
```
1. Boot. (You need to make rootfs with buildroot. Please see [here](https://zulinx86.com/notebook/kernel/environment).)
	- `-s` is a short cut for `-gdb tcp::1234` which launches a GDB server on TCP port 1234.
	- `-S` waits for its execution.
    ```
    $ qemu-system-x86_64 -boot c -m 2048M -kernel ./arch/x86/boot/bzImage -hda ${WORKDIR}/buildroot/output/images/rootfs.ext4 -append "root=/dev/sda rw console=ttyS0,115200 nokaslr" -serial stdio -display none -s -S
    ```
1. Login another terminal and run GDB.
  ```
  $ gdb vmlinux
  (gdb) target remote :1234 
  Remote debugging using :1234
  0x000000000000fff0 in exception_stacks ()

  (gdb) b start_kernel
  Breakpoint 1 at 0xffffffff82d68b5e: file init/main.c, line 928.

  (gdb) c
  Continuing.

  Breakpoint 1, start_kernel () at init/main.c:928
  928	{

  (gdb) bt
  #0  start_kernel () at init/main.c:928
  #1  0xffffffff8100011a in secondary_startup_64 () at arch/x86/kernel/head_64.S:300
  #2  0x0000000000000000 in ?? ()
  ```