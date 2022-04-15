---
layout: post
title: Environment for Kernel Development on Amazon Linux 2
---


## Environment
- AMI: Amazon Linux 2
- Instance Family: M5



## Step 1. Clone Linux Kernel Repository
Clone a repository that you want to clone from [Kernel.org git repositories](https://git.kernel.org/).  
```
$ cd ~
$ sudo yum install -y git
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```



## Step 2. Build Linux Kernel
[Minimal requirements to compile the Kernel — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/process/changes.html)

```
$ cd ~/linux
$ sudo yum install -y gcc flex bison elfutils-libelf-devel openssl-devel ncurses-devel
$ make x86_64_defconfig
$ make -j$(nproc)
...
Kernel: arch/x86/boot/bzImage is ready  (#1)
```



## Step 3. Use the Linux Kernel on Buildroot
### Step 3-1. Clone Buildroot Repository
```
$ cd ~
$ git clone git://git.buildroot.net/buildroot
```


### Step 3-2. Build Buildroot
```
$ cd ~/buildroot
$ sudo yum install -y gcc-c++ patch perl-Data-Dumper perl-ExtUtils-MakeMaker perl-Thread-Queue
$ make menuconfig
```
- Target options
	- Target Architecture: x86_64
- Filesystem images
	- ext2/3/4 root filesystem
		- ext2/3/4 variant: ext4
- System configuration
	- Network interface to configure through DHCP
		- eth0
```
$ make -j$(nproc)
```


### Step 3-3. Boot
```
$ sudo yum install -y qemu
$ qemu-system-x86_64 \
	-boot order=c \
	-m 2G \
	-kernel ~/linux/arch/x86/boot/bzImage \
	-hda ~/buildroot/output/images/rootfs.ext4 \
	-append "root=/dev/sda rw console=ttyS0,115200" \
	-serial stdio \
	-display none
```



## Step 4. Make It Easy to Read Source Code
### command alias
Append the following in `~/.bashrc`.
```
alias cgrep='grep -rn --include=*.c --exclude=*.mod.c'
alias hgrep='grep -rn --include=*.h'
alias sgrep='grep -rn --include=*.S'
alias chgrep='grep -rn --include=*.c --include=*.h --exclude=*.mod.c'
alias chsgrep='grep -rn --include=*.c --include=*.h --include=*.S --exclude=*.mod.c'

alias qk='qemu-system-x86_64 -boot order=c -m 2G -kernel ~/linux/arch/x86/boot/bzImage -hda ~/buildroot/output/images/rootfs.ext4 -append "root=/dev/sda rw console=ttyS0,115200" -serial stdio -display none'
alias qm='sudo mount -o loop ~/buildroot/output/images/rootfs.ext4 /mnt/buildroot'
alias qu='sudo umount /mnt/buildroot'
```


### ctags
```
$ cd ~/linux
$ sudo yum install -y ctag
$ make tags
$ vim -t <keyword>
```



## Misc
### Replace the Linux Kernel on Your System
```
$ cd ~/linux
$ cp /boot/config-$(uname -r) .config
$ make olddefconfig
$ make menuconfig
```
- Device Drivers
	- Network device support
		- Ethernet driver support
			- Amazon Devices
				- Elastic Network Adapter (ENA) support
```
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo grub2-mkconfig -o /boot/grub2/grub.cfg
$ sudo reboot
```



## Links
- [Linux カーネルを QEMU 上で起動する - Kuniyuki Iwashima](https://kuniyu.jp/ja/blog/2/)