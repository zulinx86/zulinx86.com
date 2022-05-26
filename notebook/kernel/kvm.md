---
layout: post
title: KVM
---



## What is KVM?
- General
	- Kernel-based Virtual Machine (KVM) is a virtualization module in the Linux kernel that allows the kernel to function as a hypervisor.
	- Using KVM, one can run multiple virtual machines on a single host.
	- KVM requires a processor with hardware virtualization extensions (such as Intel VT or AMD-V).
	- KVM hypervisor = Linux kernel + KVM module (kvm.ko) + processor specific module (kvm-intel.ko or kvm-amd.ko).
	- KVM exposes a character device `/dev/kvm`, which a userspace VMM can use to manage the guest VMs via a set of ioctls.
- KVM and QEMU
	- KVM cannot run VMs by itself.
	- To run VMs, it requires QEMU as a userspace program to provide hardware emulation (e.g., I/O devices).
	- There is one QEMU process for each guest system. QEMU is a multi-thread program, and one virtual CPU to one QEMU thread.
- Virtualization with KVM
	- A virtual machine using KVM need not run a complete operating system or emulate a full suite of hardware devices.
	- Using the KVM API, a program can run code inside a sandbox and provide arbitrary virtual hardware interfaces to that sandbox.

```
+---------------++---------------+ +-------+--------------+--------------+
|   KVM Guest   ||   KVM Guest   | | virsh | virt-install | virt-manager |
|               ||               | +-------+----------+---+--------------+
|+-------------+||+-------------+|                    |
|| Application |||| Application || +------------------v------------------+
|+-------------+||+-------------+| |               libvirt               |
||     OS      ||||     OS      || +---------+------------------+--------+
|+-------------+||+-------------+|           |                  |
|| Emulated HW |||| Emulated HW || +---------v--------++--------v--------+
|+-------------+||+-------------+| |       QEMU       ||      QEMU       |
+---------------++------^-+------+ +------------------++-------^-+-------+
               VM Entry | | VM Exit                  Emulation | | ioctl()
+-----------------------+-v------+------+----------------------+-v-------+
|             kvm.ko             <------>           /dev/kvm             |
|--------------------------------+      +--------------------------------+
|                                 Kernel                                 |
+------------------------------------------------------------------------+
|                     Hardware (Intel VT-x / AMD-V)                      |
+------------------------------------------------------------------------+
```



## Tutorial
1. Check if KVM is available.
	```
	# lsmod | grep kvm
	kvm_intel             323584  0
	kvm                   880640  1 kvm_intel
	irqbypass              16384  1 kvm
	```
1. Install packages
	```
	# dnf install qemu-kvm libvirt virt-install -y
	```
1. Start libvirtd.
	```
	# systemctl enable libvirtd
	# systemctl start libvirtd

	# systemctl status libvirtd
	● libvirtd.service - Virtualization daemon
	   Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; enabled; vendor preset: enabled)
	   Active: active (running) since Wed 2022-05-25 04:38:34 UTC; 19s ago
	     Docs: man:libvirtd(8)
	           https://libvirt.org
	```
1. Download ISO file from https://archives.fedoraproject.org/pub/archive/fedora/linux/releases/ .
	```
	# cd /var/lib/libvirt/images/
	# sudo wget https://archives.fedoraproject.org/pub/archive/fedora/linux/releases/35/Server/x86_64/iso/Fedora-Server-dvd-x86_64-35-1.2.iso
	```
1. Install Fedora on VM and go forward installation.
	```
	# virt-install \
	--virt-type kvm \
	--name fedora35 \
	--os-variant fedora35 \
	--location /var/lib/libvirt/images/Fedora-Server-dvd-x86_64-35-1.2.iso \
	--disk path=/var/lib/libvirt/images/fedora35.qcow2,size=10,format=qcow2,bus=virtio \
	--network network=default,model=virtio \
	--graphics none \
	--extra-args 'console=ttyS0,115200n8'
	```



## libvirt
- What is libvirt?
	- libvirt is an open-source project to provide a standardized API for virtualization.
	- libvirt is a C library with bindings with other languages, notably in Python, Perl, OCaml, Ruby, Java, JavaScript and PHP.
- Supported hypervisors: KVM, Xen, QEMU, LXC, VMware ESX, VirtualBox, Hyper-V, etc.
- Components
	- libvirtd: Server-side daemon providing an interface to manage VMs via libvirt API.
	- virsh: CUI client tool to use libvirt API.
	- virt-install: CUI client tool to provision new VMs.
	- virt-manager: GUI client tool to use libvirt API.
	- virt-top: CUI tool to monitor VMs.
	- virt-viewer: Tool to enable local connection.


### libvirtd
- Major features
	- VM management: Various domain lifecycle operations such as start, stop, pause, save, restore, and migrate.
	- Remote machine support: The libvirtd daemon listens on a local Unix domain socket by default. It can listen on a TCP/IP socket optionally.
	- Storage management: Management of various types of storage.
	- Network interface management: Management of physical and logical network interfaces.
	- Virtual NAT and route based networking: Management of virtual networks.
- Configuration files
	- `/etc/libvirt/libvirtd.conf`


### virsh
#### Domain Management
```
# List all VMs
virsh list --all

# Create XML and register VM
virsh define <xml>

# Create XML, register and start VM
virsh create <xml>

# Start VM
virsh start <name>

# Shutdown VM
virsh shutdown <name>

# Stop VM
virsh destroy <name>

# Delete XML and unregister VM
virsh undefine <name>

# Export XML
virsh dumpxml <name>

# Edit XML
virsh edit <name>

# Connect to console
virsh console <name>
```



## Links
- [KVM: Kernel-based Virtual Machine (v2) - LWN.net](https://lwn.net/Articles/205580/)
- [Using the KVM API - LWN.net](https://lwn.net/Articles/658511/)
- [api.rst - Documentation/virt/kvm/api.rst - Linux source code (v5.18) - Bootlin](https://elixir.bootlin.com/linux/latest/source/Documentation/virt/kvm/api.rst)
- [KVM](https://www.linux-kvm.org/page/Main_Page)
- [Kernel-based Virtual Machine - Wikipedia](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine)
- [libvirt: The virtualization API](https://libvirt.org/)
- [libvirt: Documentation](https://libvirt.org/docs.html)
