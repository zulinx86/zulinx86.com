---
title: 【Kernel】/proc/cpuinfo
emoji: "🐧"
type: "tech"
topics: ["linux", "kernel"]
published: true
---


# Kernel Code (v6.11.5)

### `struct proc_ops`

[https://elixir.bootlin.com/linux/v6.11.5/source/include/linux/proc_fs.h#L29](https://elixir.bootlin.com/linux/v6.11.5/source/include/linux/proc_fs.h#L29)
```c
struct proc_ops {
	unsigned int proc_flags;
	int	(*proc_open)(struct inode *, struct file *);
	ssize_t	(*proc_read)(struct file *, char __user *, size_t, loff_t *);
	ssize_t (*proc_read_iter)(struct kiocb *, struct iov_iter *);
	ssize_t	(*proc_write)(struct file *, const char __user *, size_t, loff_t *);
	/* mandatory unless nonseekable_open() or equivalent is used */
	loff_t	(*proc_lseek)(struct file *, loff_t, int);
	int	(*proc_release)(struct inode *, struct file *);
	__poll_t (*proc_poll)(struct file *, struct poll_table_struct *);
	long	(*proc_ioctl)(struct file *, unsigned int, unsigned long);
#ifdef CONFIG_COMPAT
	long	(*proc_compat_ioctl)(struct file *, unsigned int, unsigned long);
#endif
	int	(*proc_mmap)(struct file *, struct vm_area_struct *);
	unsigned long (*proc_get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
} __randomize_layout;
```

### `cpu_info_proc_ops`

[https://elixir.bootlin.com/linux/v6.11.5/source/fs/proc/cpuinfo.c#L15](https://elixir.bootlin.com/linux/v6.11.5/source/fs/proc/cpuinfo.c#L15)
```c
static const struct proc_ops cpuinfo_proc_ops = {
	.proc_flags	= PROC_ENTRY_PERMANENT,
	.proc_open	= cpuinfo_open,
	.proc_read_iter	= seq_read_iter,
	.proc_lseek	= seq_lseek,
	.proc_release	= seq_release,
};
```

### `proc_cpuinfo_init()`

[https://elixir.bootlin.com/linux/v6.11.5/source/fs/proc/cpuinfo.c#L23](https://elixir.bootlin.com/linux/v6.11.5/source/fs/proc/cpuinfo.c#L23)
```c
static int __init proc_cpuinfo_init(void)
{
	proc_create("cpuinfo", 0, NULL, &cpuinfo_proc_ops);
	return 0;
}
fs_initcall(proc_cpuinfo_init);
```

### `cpuinfo_open()`

[https://elixir.bootlin.com/linux/v6.11.5/source/fs/proc/cpuinfo.c#L10](https://elixir.bootlin.com/linux/v6.11.5/source/fs/proc/cpuinfo.c#L10)
```c
static int cpuinfo_open(struct inode *inode, struct file *file)
{
	return seq_open(file, &cpuinfo_op);
}
```


## arm64

### `cpuinfo_op`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/kernel/cpuinfo.c#L275](https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/kernel/cpuinfo.c#L275)
```c
const struct seq_operations cpuinfo_op = {
	.start	= c_start,
	.next	= c_next,
	.stop	= c_stop,
	.show	= c_show
};
```

### `c_show()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/kernel/cpuinfo.c#L193](https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/kernel/cpuinfo.c#L193)
```c
static int c_show(struct seq_file *m, void *v)
{
	int i, j;
	bool compat = personality(current->personality) == PER_LINUX32;

	for_each_online_cpu(i) {
		struct cpuinfo_arm64 *cpuinfo = &per_cpu(cpu_data, i);
		u32 midr = cpuinfo->reg_midr;

		/*
		 * glibc reads /proc/cpuinfo to determine the number of
		 * online processors, looking for lines beginning with
		 * "processor".  Give glibc what it expects.
		 */
		seq_printf(m, "processor\t: %d\n", i);
		if (compat)
			seq_printf(m, "model name\t: ARMv8 Processor rev %d (%s)\n",
				   MIDR_REVISION(midr), COMPAT_ELF_PLATFORM);

		seq_printf(m, "BogoMIPS\t: %lu.%02lu\n",
			   loops_per_jiffy / (500000UL/HZ),
			   loops_per_jiffy / (5000UL/HZ) % 100);

		/*
		 * Dump out the common processor features in a single line.
		 * Userspace should read the hwcaps with getauxval(AT_HWCAP)
		 * rather than attempting to parse this, but there's a body of
		 * software which does already (at least for 32-bit).
		 */
		seq_puts(m, "Features\t:");
		if (compat) {
#ifdef CONFIG_COMPAT
			for (j = 0; j < ARRAY_SIZE(compat_hwcap_str); j++) {
				if (compat_elf_hwcap & (1 << j)) {
					/*
					 * Warn once if any feature should not
					 * have been present on arm64 platform.
					 */
					if (WARN_ON_ONCE(!compat_hwcap_str[j]))
						continue;

					seq_printf(m, " %s", compat_hwcap_str[j]);
				}
			}

			for (j = 0; j < ARRAY_SIZE(compat_hwcap2_str); j++)
				if (compat_elf_hwcap2 & (1 << j))
					seq_printf(m, " %s", compat_hwcap2_str[j]);
#endif /* CONFIG_COMPAT */
		} else {
			for (j = 0; j < ARRAY_SIZE(hwcap_str); j++)
				if (cpu_have_feature(j))
					seq_printf(m, " %s", hwcap_str[j]);
		}
		seq_puts(m, "\n");

		seq_printf(m, "CPU implementer\t: 0x%02x\n",
			   MIDR_IMPLEMENTOR(midr));
		seq_printf(m, "CPU architecture: 8\n");
		seq_printf(m, "CPU variant\t: 0x%x\n", MIDR_VARIANT(midr));
		seq_printf(m, "CPU part\t: 0x%03x\n", MIDR_PARTNUM(midr));
		seq_printf(m, "CPU revision\t: %d\n\n", MIDR_REVISION(midr));
	}

	return 0;
}
```


## x86

### `cpuinfo_op`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/proc.c#L174](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/proc.c#L174)
```c
const struct seq_operations cpuinfo_op = {
	.start	= c_start,
	.next	= c_next,
	.stop	= c_stop,
	.show	= show_cpuinfo,
};
```

### `show_cpuinfo()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/proc.c#L63](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/proc.c#L63)
```c
static int show_cpuinfo(struct seq_file *m, void *v)
{
	struct cpuinfo_x86 *c = v;
	unsigned int cpu;
	int i;

	cpu = c->cpu_index;
	seq_printf(m, "processor\t: %u\n"
		   "vendor_id\t: %s\n"
		   "cpu family\t: %d\n"
		   "model\t\t: %u\n"
		   "model name\t: %s\n",
		   cpu,
		   c->x86_vendor_id[0] ? c->x86_vendor_id : "unknown",
		   c->x86,
		   c->x86_model,
		   c->x86_model_id[0] ? c->x86_model_id : "unknown");

	if (c->x86_stepping || c->cpuid_level >= 0)
		seq_printf(m, "stepping\t: %d\n", c->x86_stepping);
	else
		seq_puts(m, "stepping\t: unknown\n");
	if (c->microcode)
		seq_printf(m, "microcode\t: 0x%x\n", c->microcode);

	if (cpu_has(c, X86_FEATURE_TSC)) {
		unsigned int freq = arch_freq_get_on_cpu(cpu);

		seq_printf(m, "cpu MHz\t\t: %u.%03u\n", freq / 1000, (freq % 1000));
	}

	/* Cache size */
	if (c->x86_cache_size)
		seq_printf(m, "cache size\t: %u KB\n", c->x86_cache_size);

	show_cpuinfo_core(m, c, cpu);
	show_cpuinfo_misc(m, c);

	seq_puts(m, "flags\t\t:");
	for (i = 0; i < 32*NCAPINTS; i++)
		if (cpu_has(c, i) && x86_cap_flags[i] != NULL)
			seq_printf(m, " %s", x86_cap_flags[i]);

#ifdef CONFIG_X86_VMX_FEATURE_NAMES
	if (cpu_has(c, X86_FEATURE_VMX) && c->vmx_capability[0]) {
		seq_puts(m, "\nvmx flags\t:");
		for (i = 0; i < 32*NVMXINTS; i++) {
			if (test_bit(i, (unsigned long *)c->vmx_capability) &&
			    x86_vmx_flags[i] != NULL)
				seq_printf(m, " %s", x86_vmx_flags[i]);
		}
	}
#endif

	seq_puts(m, "\nbugs\t\t:");
	for (i = 0; i < 32*NBUGINTS; i++) {
		unsigned int bug_bit = 32*NCAPINTS + i;

		if (cpu_has_bug(c, bug_bit) && x86_bug_flags[i])
			seq_printf(m, " %s", x86_bug_flags[i]);
	}

	seq_printf(m, "\nbogomips\t: %lu.%02lu\n",
		   c->loops_per_jiffy/(500000/HZ),
		   (c->loops_per_jiffy/(5000/HZ)) % 100);

#ifdef CONFIG_X86_64
	if (c->x86_tlbsize > 0)
		seq_printf(m, "TLB size\t: %d 4K pages\n", c->x86_tlbsize);
#endif
	seq_printf(m, "clflush size\t: %u\n", c->x86_clflush_size);
	seq_printf(m, "cache_alignment\t: %d\n", c->x86_cache_alignment);
	seq_printf(m, "address sizes\t: %u bits physical, %u bits virtual\n",
		   c->x86_phys_bits, c->x86_virt_bits);

	seq_puts(m, "power management:");
	for (i = 0; i < 32; i++) {
		if (c->x86_power & (1 << i)) {
			if (i < ARRAY_SIZE(x86_power_flags) &&
			    x86_power_flags[i])
				seq_printf(m, "%s%s",
					   x86_power_flags[i][0] ? " " : "",
					   x86_power_flags[i]);
			else
				seq_printf(m, " [%d]", i);
		}
	}

	seq_puts(m, "\n\n");

	return 0;
}
```
