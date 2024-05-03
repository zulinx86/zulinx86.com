---
title: 【KVM】kvm_run.immediate_exit
emoji: "🐧"
type: "idea"
topics: ["linux", "kvm"]
published: true
---

# What is `immediate_exit`?

https://www.kernel.org/doc/html/latest/virt/kvm/api.html#the-kvm-run-structure

> Application code obtains a pointer to the kvm_run structure by mmap()ing a vcpu fd. From that point, application code can control execution by changing fields in kvm_run prior to calling the KVM_RUN ioctl, and obtain information about the reason KVM_RUN returned by looking up structure members.

> ```
> __u8 immediate_exit;
> ```
> This field is polled once when KVM_RUN starts; if non-zero, KVM_RUN exits immediately, returning -EINTR. In the common scenario where a signal is used to “kick” a VCPU out of KVM_RUN, this field can be used to avoid usage of KVM_SET_SIGNAL_MASK, which has worse scalability. Rather than blocking the signal outside KVM_RUN, userspace can set up a signal handler that sets run->immediate_exit to a non-zero value.

## Sample Code

A starting point is the following code.

```c
#include <err.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/ioctl.h>
#include <linux/kvm.h>

int main(int argc, char *argv[])
{
	int ret;

	/* Open KVM device */
	int kvm_fd = open("/dev/kvm", O_RDWR | O_CLOEXEC);
	if (kvm_fd == -1)
		err(1, "open /dev/kvm");

	/* Create VM */
	int vm_fd = ioctl(kvm_fd, KVM_CREATE_VM, 0);
	if (vm_fd == -1)
		err(1, "KVM_CREATE_VM");

	/* Allocate guest memory */
	uint8_t *guest_mem = mmap(NULL, 0x1000, PROT_READ | PROT_WRITE,
				  MAP_SHARED | MAP_ANONYMOUS, -1, 0);
	if (guest_mem == MAP_FAILED)
		err(1, "allocate geust memory");

	const uint8_t guest_code[] = {
		0xeb, 0xfe /* infinite loop: jump $-2 */
	};
	memcpy(guest_mem, guest_code, sizeof(guest_code));

	/* Map it to the second page frame (to avoid the real-mode IDT at 0) */
	struct kvm_userspace_memory_region region = {
		.slot = 0,
		.guest_phys_addr = 0x1000,
		.memory_size = 0x1000,
		.userspace_addr = (uint64_t)guest_mem,
	};
	ret = ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &region);
	if (ret == -1)
		err(1, "KVM_SET_USER_MEMORY_REGION");

	/* Create vCPU */
	int vcpu_fd = ioctl(vm_fd, KVM_CREATE_VCPU, 0);
	if (vcpu_fd == -1)
		err(1, "KVM_CREATE_VCPU");

	/* Map kvm_run */
	ret = ioctl(kvm_fd, KVM_GET_VCPU_MMAP_SIZE, 0);
	if (ret == -1)
		err(1, "KVM_GET_VCPU_MMAP_SIZE");
	size_t vcpu_mmap_size = ret;
	struct kvm_run *run = mmap(NULL, vcpu_mmap_size, PROT_READ | PROT_WRITE,
				   MAP_SHARED, vcpu_fd, 0);
	if (run == MAP_FAILED)
		err(1, "mmap vcpu");

	/* Initialize registers, instruction pointer for guest code */
	struct kvm_regs regs = {
		.rip = 0x1000,
	};
	ret = ioctl(vcpu_fd, KVM_SET_REGS, &regs);
	if (ret == -1)
		err(1, "KVM_SET_REGS");

	/* Initialize CS to point at 0 */
	struct kvm_sregs sregs;
	ret = ioctl(vcpu_fd, KVM_GET_SREGS, &sregs);
	if (ret == -1)
		err(1, "KVM_GET_SREGS");
	sregs.cs.base = 0;
	sregs.cs.selector = 0;
	ret = ioctl(vcpu_fd, KVM_SET_SREGS, &sregs);
	if (ret == -1)
		err(1, "KVM_SET_SREGS");

	/* Run guest and handle VM exits */
	while (1) {
		ret = ioctl(vcpu_fd, KVM_RUN, 0);
		if (ret == -1)
			err(1, "KVM_RUN");

		switch (run->exit_reason) {
		default:
			err(1, "exit_reason = 0x%x", run->exit_reason);
		}
	}

	close(kvm_fd);
	return 0;
}
```

The guest just loops infinitely and KVM never exits.

If you set `kvm_run.immediate_exit = 1` just before the guest loop, KVM exits with `-EINTR`.

```c
        /* Run guest and handle VM exits */
+       run->immediate_exit = 1;
        while (1) {
```
```
$ ./a.out
a.out: KVM_RUN: Interrupted system call
```

Next, let's make it more practical and set `kvm_run.immediate_exit = 1` when it receives a signal.

See the result first. Once `SIGUSR1` (10) is received, the guest loop exits.

```
$ ./a.out
pid: 49336
a.out: KVM_RUN: Interrupted system call
```
```
$ kill -10 49336
```

The main changes are:
- declare `struct kvm_run *run;` in the global scope to allow the signal handler to access it.
- show PID to send a signal to.
- change guest code to infinite loop of `HLT` to exit `KVM_RUN` and allow `KVM_RUN` to know `kvm_run.immediate_exit` changed.
- set up a signal handler on `SIGUSR1`.
- handle the exit reason for `HLT` (`KVM_EXIT_HLT`).

The full code is as follows:

```c
#include <err.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/ioctl.h>
#include <linux/kvm.h>

struct kvm_run *run;

void signal_handler(int signo)
{
	run->immediate_exit = 1;
}

int main(int argc, char *argv[])
{
	int ret;

	/* Show PID */
	pid_t pid = getpid();
	printf("pid: %d\n", pid);

	/* Open KVM device */
	int kvm_fd = open("/dev/kvm", O_RDWR | O_CLOEXEC);
	if (kvm_fd == -1)
		err(1, "open /dev/kvm");

	/* Create VM */
	int vm_fd = ioctl(kvm_fd, KVM_CREATE_VM, 0);
	if (vm_fd == -1)
		err(1, "KVM_CREATE_VM");

	/* Allocate guest memory */
	uint8_t *guest_mem = mmap(NULL, 0x1000, PROT_READ | PROT_WRITE,
				  MAP_SHARED | MAP_ANONYMOUS, -1, 0);
	if (guest_mem == MAP_FAILED)
		err(1, "allocate geust memory");

	const uint8_t guest_code[] = {
		0xf4, /* hlt */
		0xeb, 0xfd /* infinite loop: jump $-3 */
	};
	memcpy(guest_mem, guest_code, sizeof(guest_code));

	/* Map it to the second page frame (to avoid the real-mode IDT at 0) */
	struct kvm_userspace_memory_region region = {
		.slot = 0,
		.guest_phys_addr = 0x1000,
		.memory_size = 0x1000,
		.userspace_addr = (uint64_t)guest_mem,
	};
	ret = ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &region);
	if (ret == -1)
		err(1, "KVM_SET_USER_MEMORY_REGION");

	/* Create vCPU */
	int vcpu_fd = ioctl(vm_fd, KVM_CREATE_VCPU, 0);
	if (vcpu_fd == -1)
		err(1, "KVM_CREATE_VCPU");

	/* Map kvm_run */
	ret = ioctl(kvm_fd, KVM_GET_VCPU_MMAP_SIZE, 0);
	if (ret == -1)
		err(1, "KVM_GET_VCPU_MMAP_SIZE");
	size_t vcpu_mmap_size = ret;
	run = mmap(NULL, vcpu_mmap_size, PROT_READ | PROT_WRITE,
		   MAP_SHARED, vcpu_fd, 0);
	if (run == MAP_FAILED)
		err(1, "mmap vcpu");

	/* Initialize registers, instruction pointer for guest code */
	struct kvm_regs regs = {
		.rip = 0x1000,
	};
	ret = ioctl(vcpu_fd, KVM_SET_REGS, &regs);
	if (ret == -1)
		err(1, "KVM_SET_REGS");

	/* Initialize CS to point at 0 */
	struct kvm_sregs sregs;
	ret = ioctl(vcpu_fd, KVM_GET_SREGS, &sregs);
	if (ret == -1)
		err(1, "KVM_GET_SREGS");
	sregs.cs.base = 0;
	sregs.cs.selector = 0;
	ret = ioctl(vcpu_fd, KVM_SET_SREGS, &sregs);
	if (ret == -1)
		err(1, "KVM_SET_SREGS");

	/* Set up signal handler */
	struct sigaction act;
	act.sa_handler = signal_handler;
	sigemptyset(&act.sa_mask);
	act.sa_flags = 0;
	sigaction(SIGUSR1, &act, NULL);

	/* Run guest and handle VM exits */
	while (1) {
		ret = ioctl(vcpu_fd, KVM_RUN, 0);
		if (ret == -1)
			err(1, "KVM_RUN");

		switch (run->exit_reason) {
		case KVM_EXIT_HLT:
			/* Guest has halted, continue the loop */
			break;
		default:
			err(1, "exit_reason = 0x%x", run->exit_reason);
		}
	}

	close(kvm_fd);
	return 0;
}
```


# References

- [The Definitive KVM (Kernel-based Virtual Machine) API Documentation — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/virt/kvm/api.html#the-kvm-run-structure)
- [[13/17] KVM: use KVM_CAP_IMMEDIATE_EXIT - Patchwork](https://patchwork.ozlabs.org/project/qemu-devel/patch/20170227124551.8673-14-pbonzini@redhat.com/)
- [Using the KVM API [LWN.net]](https://lwn.net/Articles/658511/)
