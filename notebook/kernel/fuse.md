---
layout: post
title: FUSE (Filesystem in USErspace)
---

## What is FUSE?
- FUSE (Filesystem in USErspace) is a software interface that lets non-privileged uses create their own file systems without editing kernel code.
- This is achieved by running filesystem code in user space while the FUSE module provides only a bridge to the actual kernel interfaces.



## How FUSE works
- VFS (Virtual File System)
	- abstraction layer for file system which provides the interface for file operations (e.g. `open`, `stat`, `read`).
- FUSE kernel module (fuse.ko)
	- kernel module for FUSE which interacts with user space via `/dev/fuse`.
- `/dev/fuse`
	- device file which is in charge of interface between kernel space and user space.
- libfuse
	- library for FUSE which reads/writes `/dev/fuse` and hooks the corresponding function of filesystem daemon based on the request.
- Filesystem daemon
	- daemon process running in user space which handles file system operations actually.

```
                              +-----------------------+
                              |   Filesystem daemon   |
                              +-----------+-----------+
                                          |
   +-----------------------+  +-----------+-----------+
   | Application (e.g. ls) |  |        libfuse        |
   +------------+----------+  +-----------+-----------+
                |                         |
user space      |             +-----------+-----------+
----------------|-------------|      /dev/fuse        |--------
kernel space    |             +-----------------------+
                |                         |
   +------------+----------+  +-----------+-----------+
   |                       +--+          FUSE         |
   |                       |  +-----------------------+
   |                       |  +-----------------------+
   |                       |  |          NFS          |
   |                       |  +-----------------------+
   |          VFS          |  +-----------------------+
   |                       |  |          EXT4         |
   |                       |  +-----------------------+
   |                       |  +-----------------------+
   |                       |  |          ...          |
   +-----------------------+  +-----------------------+
```


## Links
- [FUSE — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/filesystems/fuse.html)
- [Filesystem in Userspace - Wikipedia](https://en.wikipedia.org/wiki/Filesystem_in_Userspace)
- [Filesystem in Userspace (FUSE) のカーネルとデーモン間の通信 - Qiita](https://qiita.com/tkusumi/items/6dc204ba964264c72a9a)
- [入門 FUSE](https://blog.ssrf.in/post/fuse-tutorial/)
