---
layout: post
title: CPU Virtualization
---


## Resources to be Virtualized
- CPU (sensitive instructions) (described in this note)
- Memory
- I/O devices (disk and network)
- Interrupt controler
- Timer
- Motherboard
- Device bus
- BIOS



## Virtual Machine Monitor (VMM) / Hypervisor
- A virtual machine is taken to be an efficient, isolated duplicate of the real machine.
- A virtual machine monitor (VMM) allows multiple virtual machines (VMs)to run concurrently on a single hardware.
- Types of hypervisor
	- Hypervisors are mainly categoried as either type 1 or type 2.
		- Type 1: bare-metal, embedded, or native hypervisor
			- The hypervisor/VMM runs directly on top of the hardware.
		- Type 2: hosted hypervisor
			- There is an OS present and the hypervisor/VMM runs as an application on it.
	- But there is no clear or standard definition of type 1 and type 2 hypervisors.
- A VMM has three essential properties:
	1. Equivalence
		- The VMM provides an environment for programs which is essentially identical with the original machine.
		- A program running under the VMM should exhibit a behavior essentially identical to that demonstrated when running on the original machine.
	1. Efficiency
		- Programs run in this environment show at worst only monir decreases in speed.
		- A statistically dominant subset of the virtual processor's instructions must be executed directy by the real processor without no software intervention by the VMM.
	1. Resource Control
		- The VMM is in complete control of system resources.
			- It is not possible for a program under the VMM to access any resource not explicitly allocated to it.
			- It is possible under certain circumstances for the VMM to regain control of resources already allocated.
- Difference between VMM and hypervisor
	- Some literatures treat these terms as synonyms, others ascribe a narrower role to the VMM.
		- A VMM is an entity specifically responsible for virtualizing a given acrhitecture, including the instruction set, memory, interrupts, and basic I/O operations.
		- A hypervisor combines an operating system (OS) with a VMM.
	- VMware's vSphere ESX hypervisor:
		- It is comprised of the vmkernel and a VMM.
		- The vmkernel loads the VMM to run a VM and the VM executes directly on top of the VMM.
		- VMware's products create a separate instance of the VMM for each running VM.
		- The vmkernel manages host-wide tasks such as scheduling, while VMMs manage per-VM tasks.



## Virtualization Theorem
- Popek and Goldberg describe the characteristics that the instruction set architecture (ISA) of the physical processor must possess in order to run VMMs which possesses the above VMM's properties.
- Classification of instructions
	- Privileged vs. Unprivileged instructions
		- Privileged instructions: trap if the processor is in user mode and do not trap if it is in supervisor mode.
		- Unprivileged instructions: do not trap regardless of processor mode.
	- Sensitive vs. Innocuous instructions
		- Control sensitive instructions: attempt to change the amount of resources available or affect the processor mode.
		- Bahavior sensitive instructions: whose behavior or result depends on the configuration of resources.
- Theorem
	- A virtual machine monitor may be constructed if the set of sensitive instructions is a subset of the set of privileged instructions.
		- Intuitively, it states that to build a VMM it is sufficient that all sensitive instructions always trap and pass control to the VMM.
		- Non-privileged instructions must instead be executed natively for efficiency.



## Virtualizability of Intel Pentium
- Robin and Irvine report an analysis of the virtualizability of all of the approximately 250 instructions of the Intel Pentium platform and address its ability to support a VMM.
- They reveal that the Pentium instruction set contains 18 sensitive, unprivilege instructions that violate the requirement of the virtualization thoerem.
	- SGDT, SIDT, SLDT
		- These instructions store special registers value (GDTR, IDTR, and LDTR) into some location, which means that it cannot  protect the sensitive registers from reading by unprivileged software.
		- Since the Intel processor only has one GDTR, IDTR, and LDTR, the VMM must provide each VM with its own virtual set of IDTR, LDTR, and GDTR.
	- SMSW
		- The SMSW instruction stores the machine status word (bits 0 through 15 of CR0) into a general purpose register or memory location.
		- Bits 0 through 5 of CR0 contain system flags that control the operating mode and state of the processor.
			- For example, bit 0 is protection enable (PE) that enable protected mode when set and real mode when clear.
			- Consider the following scenario: An operating system of a virtual machine (VMOS) is running in real mode within the virtual environment created by a VMM running in protected mode.
			- If the VMOS checked the MSW to see if it was in real mode, it would incorrectly see that the PE bit is set, which means that the machine is in protected mode.
	- PUSHF, POPF
		- The PUSHF/PUSHFD instruction pushes the EFLAGS onto the stack and the POPF/POPFD instrution pops from the top of the stack and stores the value in the EFLAGS.
		- The PUSHF/PUSHFD allow the contents of the EFLAGS to be examined.
		- The POPF/POPFD allow modification of certain bits in the EFLAGS that control the operating mode and state of the processor.
	- LAR, LSL, VERR, VERW
		- The LAR instruction loads access rights from a segment descriptor into a general purpose register.
		- The LSL instruction loads the unscrambled segment limit from the segment descriptor into a general purpose register.
		- The VERR and VERW instructions verify whether a code or data segment is readable or writable from the current privilege level.
		- This is a problem becuase a VM normally does not execute at CPL = 0, but normally at CPL = 3, so that all privileged instrutions will cause traps that can be handled by the VMM. However, most operating systems assume that they are operating at the highest privilege level and that they can access any segment descriptor. Therefore, if a VMOS running at CPL = 3 uses these instructions to exemine a segment descriptor, it is likely that the instruction will not execute properly.
	- POP
		- The POP instruction loads a value from the top of the stack to a general-purpose register, memory location, or segment register.
		- A value that is loaded into a segment register must be a valid segment selector. This privilege level checks could cause unexpected results for a VMOS that is running at CPL = 3 but assumes that it's running at CPL = 0.
	- PUSH
		- The PUSH instruction allows a general-purpose register, memory location, an immediate value, or a segment register to be pushed onto the stack.
		- Suppose that a process assuming that it's running in CPL = 0 pushes the CS to the stack. It then examines the contents of the CS on the stack to check its CPL. Upon finding that its CPL is not 0, the process may halt.
	- CALL, JMP, INT n, RET
		- Task switches and far calls to different privilege levels are problem because they involve the CPL, DPL, and RPL.
		- Since the VM normally operates at user level (CPL = 3), these privilege level checks will not work correctly when a VMOS tries to access.
	- STR
		- The STR instruction stores the segment selector from the task register into a general-purpose register or memory location, which points to the task state segment of the currently executing task.
		- If a VM running at CPL = RPL = 3 uses STR to store the contents of the task register and then examines the information, it will find that it is not running at thr privilege level at which it expects to run.
	- MOV
		- Two variants of the MOV instruction prevent Intel processor virtualization.
			- Store control registers.
				- The MOV opcode that stores segment registers allows all six of the segment registers to be stored to either a general-purpose register or to a memory location.
				- Since the CS and SS registers both contain the CPL in bits 0 and 1, a task could find that it is not operating at the expected privilege level.
			- Load control registers.
				- The MOV opcode that loads segment registers does offer some protection because it does not allow the CS register to be loaded at all.
				- However, if the task tries to load the SS register, several privilege level cheks occur that become a problem when the VM is not operating at the privilege level at which a VMOS is expecting.
- The introduction of the hardware virtualization extensions (AMD-V and Intel VT-x) allows x86 processors to meet the virtualization requirements.



## CPU Virtualization
### Full Emulation by Software
- All instructions of guest VMs are interpretted and emulated by a software.
- Procs
	- It can emulate virtual CPUs whose ISA completely different from that of the physical CPU, since all instructions are emulated by software.
- Cons
	- It is extremely slow because of the overhead of emulation.


### Trap-and-Emulate (No Hardware Support)
- All instructions of guest VMs can be executed directly on the physical CPU.
- On the other hand, the hypervisor configures the CPU to trap all potentially unsafe instructions (called sensitive instructions).
	- When the guest VM attempts to read or modify privileged state, the processor generates a trap that transfers control to the hypervisor.
	- Once the hypervisor has received a trap, it inspects the instruction and emulate it in a safe way.
	- The guest VM resumes its exeuction.
- Pros
	- It is efficient because almost all guest codes are executed directly on the physical CPU.
- Cons
	- It is available if and only if the set of sensitive instructions is a subset of the set of privileged instructions.
		- The ISA of Intel Pentium does not meet this condition.


### Binary Translation
- A x86 CPU can be virtualized by using a combination of binary translation and direct execution techniques.
	- The binary translator examines the guest kernel code before it runs, finds the sensitive but unprivileged instructions, translates them into new sequences of safe instructions that have the intended effect on the virtual hardware.
	- Meanwhile, guest application code is directly executed on the processor for high performance virtualization.
	- The VMM is running in ring 0, the guest kernel code is running in ring 1, and the guest application code is running in ring 3.
- Pros
	- The guest OS is not aware that it is being virtualized and requires no modification.
	- It is more efficient than full emulation.
- Cons
	- It is complicated and still very slow because of the overhead of interpretation and translation.


### Paravirtualization (PV)
- In PV, the guest OS is actually aware that it is being run in a virtual environment.
- The hypervisor provides efficient [hypercalls](http://xenbits.xen.org/docs/unstable/hypercall/index.html), and the guest OS uses drivers and modified kernel to use the hypercalls.
- Characteristics
	- Hypercalls for sensitive instructions
	- PV drivers for disk and network I/O
	- Interrupt controller & timers
	- Guest boots directly to the kernel without 16 bit mode or BIOS.
	- Virtualized page tables
- PV was originally introduced in Xen.
	- Xen is the hypervisor/VMM.
	- Dom0 is the control (privileged) domain that interacts with Xen and has access to the physical hardware.
	- DomU is the guest (unprivileged) domain that runs guest OS and application.
- PV drivers for I/O devices
	- The front-end driver is running on domU and the back-end driver is running on dom0.
		- For network I/O, netfront and netback.
		- For disk I/O, blockfront and blockback.
	- The front-end and back-end drivers talk via a ring buffer.
		- PV drivers use a ring buffer in a shared memory segment that is mapped to domU and dom0.
	- This shows better performance than I/O device emulation because PV drivers are much lighter than emulation of hardwares.
	- Request path
		1. An I/O request from domU is passed to dom0 via PV drivers.
		1. Dom0 issues it to the physical hardware using drivers included in dom0 OS.
- Access to the page tables was paravirtualized for memory virtualization.
- Interrupts and timers are paravirtualized.
- There is no emulated motherboard or device bus.
- Guests boot directly into the kernel in the mode the kernel wishes to run in, without needing to start in 16-bit mode or go through a BIOS.
- Pros
	- It provides lower virtualization overhead (but the performance advantage might depend on its workload).
	- PV drivers has better I/O performance than I/O device emulation.
- Cons
	- It requires modification of the guest OS.


### Hardware-assisted Virtual Machine (HVM)
- Thanks to hardware extensions (Intel VT-x and AMD-V), HVM can virtualize CPUs without binary translation and modification of guest kernel.
	- It creates privilege pseudo ring -1 that the hypervisor lives in, allowing the guest kernel to reside in ring 0 and guest application to reside in ring 3 in x86.
	- More precisely, it introduces new execution modes: VMX root mode and VMX non-root mode.
		- `VMLAUNCH` / `VMRESUME` instructions switches to VMX non-root mode, called "VM Entry".
		- Sensitive instructions in guest VM switches to VMX root mode, called "VM Exit".
- Hybrid of PV and HVM
	- (Genuine) HVM
		- The genuine HVM has the only hardware extension for CPU.
		- So other hardwares (like disk, network, and so on) still needs to be virtualized.
	- HVM with PV drivers
		- Virtualzation of network and disk (that is, emulating a full PCI card with MMIO registers and so on) is really unnecessarily complicated.
		- Because nearly all modern operating systems have ways to load third-party device drivers, it's a fairly obvious step to create disk and network drivers to use the paravirtualized interfaces.
		- It's an incremental step toward higher performance.
	- HVM with PVHVM drivers (called as "PVHVM")
		- It still has a number of things that are unncessarily inefficient.
		- Many of the paravirtualized interfaces for interrupts, timers and so on are actually available for guest VMs running in HVM mode.
		- Stefano Stabellini wrote some patches for the Linux kernel that allowed Linux to switch from using the emulated interrupt controllers and timers to the paravirtualized interrupt controllers and timers.



## Open Source Virtualization Projects
### Xen
- The core components are as follows:
	- Xen hypervisor
		- The integral part of Xen that handles intercommunication between the physical hardawre and virtual machines.
		- It handles all interrupts, times, CPU and memory requests, and hardware interaction.
	- Dom0 (privilged domain)
		- Xen's control domain that controls a virtual machines' environment.
	- DomUs (unprivileged domains)
		- Virtual machines running on Xen.

```
+---------------++---------------+     +---------------+
|     Dom0      ||     DomU      |     |     DomU      |
|+-------------+||+-------------+|     |+-------------+|
|| Application |||| Application ||     || Application ||
|+-------------+||+-------------+| ... |+-------------+|
||  Guest OS   ||||  Guest OS   ||     ||  Guest OS   ||
|| +--------+  ||||             ||     ||             ||
|| | Driver |  ||||             ||     ||             ||
|| +---^----+  ||||             ||     ||             ||
|+-----|-------+||+-------------+|     |+-------------+|
+------|--------++---------------+     +---------------+
+------|-----------------------------------------------+
|      |                Hypervisor                     |
+------|-----------------------------------------------+
|      v                 Hardware                      |
|     I/O                                              |
+------------------------------------------------------+
```


### KVM
- Kernel-based Virtual Machine (KVM) is a virtualization module in the Linux kernel that allows the kernel to function as a hypervisor.
- Using KVM, one can run multiple virtual machines on a single host.
- KVM requires a processor with hardware virtualization extensions (such as Intel VT or AMD-V).
- KVM hypervisor = Linux kernel + KVM module (kvm.ko) + processor specific module (kvm-intel.ko or kvm-amd.ko).
- Please see [kvm](https://zulinx86/notebook/kernel/kvm) page.

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



## Links
- [Popek and Goldberg virtualization requirements - Wikipedia](https://en.wikipedia.org/wiki/Popek_and_Goldberg_virtualization_requirements)
- [Formal Requirements for Virtualizable Third Generation Architectures](https://courses.cs.washington.edu/courses/cse548/08wi/papers/p412-popek.pdf)
- [Analysis of the Intel Pentium's Ability to Support a Secure Virtual Machine Monitor](https://www.usenix.org/legacy/events/sec2000/full_papers/robin/robin.pdf)
- [The Evolution of an x86 Virtual Machine Monitor](https://course.ece.cmu.edu/~ece845/sp18/docs/vmware-evolution.pdf)
- [Understanding Full Virtualization, Paravirtualization, and Hardware Assist](https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/techpaper/VMware_paravirtualization.pdf)
- [AWS EC2 Virtualization 2017: Introducing Nitro](https://www.brendangregg.com/blog/2017-11-29/aws-ec2-virtualization-2017.html)
- [Understanding the Virtualization Spectrum - Xen](https://wiki.xenproject.org/wiki/Understanding_the_Virtualization_Spectrum)
- [Xen - Interface manual](http://www-archive.xenproject.org/files/xen_interface.pdf)
