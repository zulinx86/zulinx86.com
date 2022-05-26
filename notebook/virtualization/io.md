---
layout: post
title: I/O Virtualization
---


## I/O Virtualization
- Software Emulation (e.g. QEMU)
- Paravirtualization (e.g. VirtIO)
- PCI Passthrough / Single Root I/O Virtualization (SR-IOV)
- IOMMU (e.g. Intel VT-d, AMD IOV)



## Knowledge of Device I/O
- Port-based I/O
	- Devices are mapped into the I/O space.
	- Special instructions (IN, OUT) are used to access the devices.
- Memory-mapped I/O
	- Deviecs are mapped into the memory space.
	- Normal instructions (LOAD, STORE) are used to access the devices.
	- PCI devices often provide a memory-mapped I/O space (e.g., NVMe, ENA).



## VirtIO
- VirtIO is a standardized paravirtualized interface which allows virtual machines access to simplified "virtual" devices.
- Access the virtual devices through VirtIO on a guest VM improves performane over emulated devices, as it requires only the minimum setup and configuration needed to send and receive data.
- VirtIO was originally developed by Rusty Russell for his own virtualization solution called lguest.
- The communication between the front-end drivers (implemented in the guest operating system) and the back-end drivers (implmented in the hypervisor) uses virtual queue interface implemented as rings.


### PCI
- VirtIO devices appear, to the guest VM, to be normal PCI devices with a specific Vendor ID and Device ID.
- All VirtIO devices have a Vendor ID of 0x1AF4, and have a Device ID between 0x1000 and 0x103F.
- The type of VirtIO device (network device, block device, etc.) can be determined by the Subsystem ID.

- [VirtIO](http://virtio.expert/)
- [Virtual I/O Device (VIRTIO) Version 1.0](http://docs.oasis-open.org/virtio/virtio/v1.0/virtio-v1.0.html)
- [Virtio - OSDev Wiki](https://wiki.osdev.org/Virtio)
- [Virtio - KVM](https://www.linux-kvm.org/page/Virtio)
- [Virtio: An I/O virtualization framework for Linux - IBM Developer](https://developer.ibm.com/articles/l-virtio/)



## PCI Passthrough / Single Route I/O Virtualization (SR-IOV)
- PCI devices have three memory spaces:
	- PCI configuration space
		- It includes configuration information (vendor ID, device ID, base address registers for PCI I/O space and PCI memory space, etc).
		- Access to this is done via CONFIG_ADDRESS and CONFIG_DATA registers.
	- PCI I/O space
		- This is mapped to I/O space.
		- Access to this is done via IN and OUT instructions.
	- PCI memory space
		- This is mapped to physical address space.
		- Access to this is done via normal memory access.
- PCI Paththrough
	- PCI configuration space
		- Access to CONFIG_ADDRESS and CONFIG_DATA registers are trapped by VM Exit with reason 30 (I/O instruction).
	- PCI I/O space
		- IN and OUT instructions are trapped by VM Exit with reason 30 (I/O instruction).
	- PCI memory space
		- Shadow Paging
			- The present bits of memory pages on shadow page tables are set to 0.
			- Accesses to the page from guest VM are trapped by VM Exit with reason 0 (exception or non-maskable interrupt).
		- EPT
			- Accesses to memory pages mapped into devices are trapped by VM Exit with reason 48 (EPT violation).
		- IOMMU
			- Described later.
	- Interrupt
		- Interrupts from real PCI devices are trapped by VM Exit with reason 1 (external interrupt).
- Single Root I/O Virtualization (SR-IOV)
	- SR-IOV can allow a single hardware (physical function) to appear as multiple, isolated, physical devices (virtual functions) to guest VMs.
	- Components
		- Physical Function (PF)
			- Full PCIe device including the SR-IOV capabilities (e.g., assigning VFs).
			- It is discovered, managed, and configured as a normal PCI device.
		- Virtual Function (VF)
			- Simple PCIe function that only handles I/O operations.
			- Each VF is derived from a PF.
	- Requirements
		- SR-IOV capable hardware (e.g., Intel 82599, ENA)
		- Support of Intel VT-d or AMD IOV which enables DMA in virtualized environment.


## IOMMU (Intel VT-d / AMD IOV)
- I/O devices use the machine memory address for DMA. On the other hand, the guest VM specifies the physical memory address for DMA to I/O devices. However, PCI devices are not aware that the guest VM is virtualized, so they use the physical memory address as the machine memory address, which leads to memory corruption.
- This memory corruption can be avoided if the hypervisor intervenes in the I/O operation to apply this memory translation. However, this approach incurs a delay in the I/O operation.
- IOMMU provides address conversion for I/O devices. With IOMMU, the physical memory address on the guest VM can be specified as data transfer destinations from I/O devices, i.e., data can be directly transferred to a guest VM.
- IOMMU exists between the main memory and I/O devices and translates device-visial virtual addresses into physical memory addresses.



## Links
- [ハイパーバイザの作り方～ちゃんと理解する仮想化技術～ 第１６回 PCIパススルーその２「VT-dの詳細」](https://syuu1228.github.io/howto_implement_hypervisor/part16.html)