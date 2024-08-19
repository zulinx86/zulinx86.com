---
title: Kernel 開発環境の構築 on Ubuntu
emoji: "🐧"
type: "tech"
topics: ["linux", "kernel"]
published: true
---

# 環境

- CPU: x86_64
- OS: Ubuntu 24.04

# Linux kernel のビルド

```
$ cd ${WORKDIR}
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
$ sudo apt install make gcc flex bison libelf-dev libssl-dev -y
$ cd linux
$ make defconfig
$ make -j$(nproc)
```

# Buildroot のビルド

```
$ cd ${WORKDIR}
$ git clone git://git.buildroot.net/buildroot
$ sudo apt install libncurses-dev g++ unzip bzip2 -y
$ cd buildroot
$ make menuconfig
$ make -j$(nproc)
```

- Target options
    - Target Architecture
        - "x86_64" を選択
- Filesystem images
    - "ext2/3/4 root filesystem" にチェック
        - ext2/3/4 variant
            - "ext4" を選択

# QEMU で VM の起動

```
$ sudo apt install qemu-system-x86 -y
$ qemu-system-x86_64 \
    -boot c \
    -m 8G \
    -kernel ${WORKDIR}/linux/arch/x86_64/boot/bzImage \
    -drive file=${WORKDIR}/buildroot/output/images/rootfs.ext4,format=raw \
    -append "root=/dev/sda console=ttyS0,115200" \
    -nographic
```
```
[    0.000000] Linux version 6.11.0-rc3-00013-g6b0f8db921ab (ubuntu@ip-172-31-79-48) (gcc (Ubuntu 13.2.0-23ubuntu4) 13.4
[    0.000000] Command line: root=/dev/sda console=ttyS0,115200
...
Welcome to Buildroot
buildroot login: root
#
```
