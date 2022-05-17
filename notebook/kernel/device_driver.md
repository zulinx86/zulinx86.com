---
layout: post
title: Device Driver
---


## "Hello World" Module
```
$ sudo yum install kernel-devel
$ mkdir hello
$ cd hello
```
```
$ cat Makefile 
obj-m := hello.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
```
$ cat hello.c
#include <linux/init.h>
#include <linux/module.h>

MODULE_AUTHOR("zulinx86");
MODULE_DESCRIPTION("Hello world module");
MODULE_LICENSE("Dual BSD/GPL");

static int __init hello_init(void)
{
	printk(KERN_ALERT "Hello, world!\n");
	return 0;
}

static void __exit hello_exit(void)
{
	printk(KERN_ALERT "Goodbye, world!\n");
}

module_init(hello_init);
module_exit(hello_exit);
```
```
$ make
$ sudo insmod hello.ko
$ sudo rmmod hello
$ sudo tail /var/log/messages
...
May 17 13:58:10 ip-172-31-94-46 kernel: hello: loading out-of-tree module taints kernel.
May 17 13:58:10 ip-172-31-94-46 kernel: hello: module verification failed: signature and/or required key missing - tainting kernel
May 17 13:58:10 ip-172-31-94-46 kernel: Hello, world!
May 17 13:58:29 ip-172-31-94-46 kernel: Goodbye, world!
```