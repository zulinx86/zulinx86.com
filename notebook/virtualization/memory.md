---
layout: post
title: Memory Virtualization
---


## Memory Virtualization
- Memory virtualization involves sharing the physical system memory and dynamically allocating it to virtual machines.
- Memory virtualization for virtual machines is very similar to the virtual memory support provided by modern operating system.
	- Applications see a contiguous address space that is not necessary tied to the underlying physical memory.
	- The operating system keeps mappings of virtual page numbers to physical page numbers stored in page tables.
	- All modern x86 CPUs include a memory management unit (MMU) and a translation lookaside buffer (TLB) to optimize virtual memory performance.
- To run multiple virtual machines on a single hardware, another level of memory virtualization is required.
	- In other words, it has to virtualize the MMU to support the guest OS.
	- The guest OS continues to control the mapping of virtual addresses to the guest memory physical addresses, but the guest OS cannot have direct access to the actual machine memory.
	- The VMM is responsible for mapping guest physical memory to the actual memory.


### Shadow Page Table or Shadow Paging - Software-based MMU Virtualization
- The VMM uses shadow page tables to accelerate the memory address translation.
	- Virtual memory (virtual memory in the guest VM) or logical page numbers (LPNs)
	- Physical memory (physical memory in the guest VM) or physical page numebrs (PPNs)
	- Machine memory (physical memory in the host) or machine page numbers (MPNs)
- In shadow paging, the VMM maintains PPN-to-MPN mappings in its internal data structures and stores LPN-to-MPN direct mappings in shadow page tables that are exposed to the MMU hardware in order to avoid the two-level translation on every access.
- In order to provide transparent MMU virtualization, the VMM intercepts guest page table updates to keep the shadow page tables coherent with the guest page tables.
- This causes virtualization overhead when the guest VM updates its page tables.


### Second Level Address Translation (SLAT) or Nested Paging - Hardware-assisted MMU Virtualization
- SLAT for AMD is Rapid Virtualization Indexing (RVI). SLAT for Intel is Extended Page Table (EPT).
- SLAT is a hardware extension for MMU virtualization.
- This removes much of the overhead incurred to keep the shadow page tables up-to-date.
- Under SLAT, the guest VM continues to maintain LPN-to-PPN mappings in the guest page tables, and the VMM maintains PPN-to-MPN mappings in additional level of page tables, called nested page tables. Both the guest page tables and the nested page tables are exposed to the MMU hardware.
- When a logical address is accessed, the hardware walks the guest page tables to determine the corresponding PPN same as in the case of native execution, and the harware also walks the nested page tables to determine the corresponding MPN.
- However, the extra operation also increases the cost of a page walk, thereby impacting the performance of applications that stress TLB. This cost can be reduced by using large pages, thus reducing the stress on the TLB.



## Memory Ballooning
- Memory ballooning is a technique used to eliminate the need to overprovision memory used by a VM.
- To implement it, the virtual machine's kernel implements a "balloon driver" which allocates unused memory within the reserved memory pool (called "balloon") so that the VMM can reclaim it.



## Links
- [Performance Evaluation of Intel EPT Hardware Assist](https://www.vmware.com/pdf/Perf_ESX_Intel-EPT-eval.pdf)
- [Virtualize-Memory](https://cseweb.ucsd.edu/~yiying/cse291j-winter20/reading/Virtualize-Memory.pdf)
- [Hypervisor From Scratch – Part 4: Address Translation Using Extended Page Table (EPT) | Rayanfam Blog](https://rayanfam.com/topics/hypervisor-from-scratch-part-4/)
