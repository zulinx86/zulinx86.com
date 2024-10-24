---
title: 【Security】Spectulative Store Bypass for x86
emoji: "🛡️"
type: "idea"
topics: ["security"]
published: true
---

# General Information

- Name: Speculative Store Bypass (SSB), Spectre-V4, Variant 4, Spectre Next Generation, Spectre-NG 
- CVE: CVE-2018-3639, INTEL-SA-00115
- Disclosure Date: 2018-03-21



# Latest Kernel Code (v6.11.4)

## Sysfs

- Access to `/sys/devices/system/cpu/vulnerabilities/spec_store_bypass` is handled by `cpu_show_spec_store_bypass()`.
- It returns one of the following four values depending on `ssb_mode` value:
    - `ssb_mode == SPEC_STORE_BYPASS_NONE` => "Vulnerable"
    - `ssb_mode == SPEC_STORE_BYPASS_DISABLE` => "Mitigation: Speculative Store Bypass disabled"
    - `ssb_mode == SPEC_STORE_BYPASS_PRCTL` => "Mitigation: Speculative Store Bypass disabled via prctl"
    - `ssb_mode == SPEC_STORE_BYPASS_SECCOMP` => "Mitigation: Speculative Store Bypass disabled via prctl and seccomp"

### `cpu_show_spec_store_bypass()`

[https://elixir.bootlin.com/linux/v6.11.4/source/drivers/base/cpu.c#L606](https://elixir.bootlin.com/linux/v6.11.4/source/drivers/base/cpu.c#L606)
```c
static DEVICE_ATTR(spec_store_bypass, 0444, cpu_show_spec_store_bypass, NULL);
```
The sysfs file is read only and handled by `cpu_show_spec_store_bypass()`.

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2996](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2996)
```c
ssize_t cpu_show_spec_store_bypass(struct device *dev, struct device_attribute *attr, char *buf)
{
	return cpu_show_common(dev, attr, buf, X86_BUG_SPEC_STORE_BYPASS);
}
```

### `cpu_show_common()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2916](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2916)
```c
static ssize_t cpu_show_common(struct device *dev, struct device_attribute *attr,
			       char *buf, unsigned int bug)
{
	if (!boot_cpu_has_bug(bug))
		return sysfs_emit(buf, "Not affected\n");

	switch (bug) {
// snipped
	case X86_BUG_SPEC_STORE_BYPASS:
		return sysfs_emit(buf, "%s\n", ssb_strings[ssb_mode]);
// snipped
	}

	return sysfs_emit(buf, "Vulnerable\n");
}
```

### `ssb_strings[]`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2021](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2021)
```c
static const char * const ssb_strings[] = {
	[SPEC_STORE_BYPASS_NONE]	= "Vulnerable",
	[SPEC_STORE_BYPASS_DISABLE]	= "Mitigation: Speculative Store Bypass disabled",
	[SPEC_STORE_BYPASS_PRCTL]	= "Mitigation: Speculative Store Bypass disabled via prctl",
	[SPEC_STORE_BYPASS_SECCOMP]	= "Mitigation: Speculative Store Bypass disabled via prctl and seccomp",
};
```

### `enum ssb_mitigation`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/nospec-branch.h#L513](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/nospec-branch.h#L513)
```c
/* The Speculative Store Bypass disable variants */
enum ssb_mitigation {
	SPEC_STORE_BYPASS_NONE,
	SPEC_STORE_BYPASS_DISABLE,
	SPEC_STORE_BYPASS_PRCTL,
	SPEC_STORE_BYPASS_SECCOMP,
};
```

## Vulnerability Detection

### Summary

- The processor is considered vulnerable if all the following conditions are met:
    - It is not listed in the whitelist.
    - SSB_NO is not set
        - Intel: IA32_ARCH_CAPABILITIES.SSB_NO[bit 4]
        - AMD: CPUID.80000008h:EBX.SSB_NO[bit 26]

### `X86_BUG_SPEC_STORE_BYPASS`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L506](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L506)
```c
#define X86_BUG_SPEC_STORE_BYPASS	X86_BUG(17) /* "spec_store_bypass" CPU is affected by speculative store bypass attack */
```

### `cpu_set_bug_bits()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/common.c#L1340](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/common.c#L1340)
```c
static void __init cpu_set_bug_bits(struct cpuinfo_x86 *c)
{
	u64 x86_arch_cap_msr = x86_read_arch_cap_msr();
// snipped
	if (!cpu_matches(cpu_vuln_whitelist, NO_SSB) &&
	    !(x86_arch_cap_msr & ARCH_CAP_SSB_NO) &&
	   !cpu_has(c, X86_FEATURE_AMD_SSB_NO))
		setup_force_cpu_bug(X86_BUG_SPEC_STORE_BYPASS);
// snipped
}
```

### `cpu_vuln_whitelist[]` / `NO_SSB`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/common.c#L1139](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/common.c#L1139)
```c
static const __initconst struct x86_cpu_id cpu_vuln_whitelist[] = {
// snipped
	VULNWL_INTEL(INTEL_ATOM_SILVERMONT,	NO_SSB | NO_L1TF | MSBDS_ONLY | NO_SWAPGS | NO_ITLB_MULTIHIT),
	VULNWL_INTEL(INTEL_ATOM_SILVERMONT_D,	NO_SSB | NO_L1TF | MSBDS_ONLY | NO_SWAPGS | NO_ITLB_MULTIHIT),
	VULNWL_INTEL(INTEL_ATOM_SILVERMONT_MID,	NO_SSB | NO_L1TF | MSBDS_ONLY | NO_SWAPGS | NO_ITLB_MULTIHIT),
	VULNWL_INTEL(INTEL_ATOM_AIRMONT,	NO_SSB | NO_L1TF | MSBDS_ONLY | NO_SWAPGS | NO_ITLB_MULTIHIT),
	VULNWL_INTEL(INTEL_XEON_PHI_KNL,	NO_SSB | NO_L1TF | MSBDS_ONLY | NO_SWAPGS | NO_ITLB_MULTIHIT),
	VULNWL_INTEL(INTEL_XEON_PHI_KNM,	NO_SSB | NO_L1TF | MSBDS_ONLY | NO_SWAPGS | NO_ITLB_MULTIHIT),

	VULNWL_INTEL(INTEL_CORE_YONAH,		NO_SSB),

	VULNWL_INTEL(INTEL_ATOM_AIRMONT_MID,	NO_SSB | NO_L1TF | NO_SWAPGS | NO_ITLB_MULTIHIT | MSBDS_ONLY),
	VULNWL_INTEL(INTEL_ATOM_AIRMONT_NP,	NO_SSB | NO_L1TF | NO_SWAPGS | NO_ITLB_MULTIHIT),
// snipped
	/* AMD Family 0xf - 0x12 */
	VULNWL_AMD(0x0f,	NO_MELTDOWN | NO_SSB | NO_L1TF | NO_MDS | NO_SWAPGS | NO_ITLB_MULTIHIT | NO_MMIO | NO_BHI),
	VULNWL_AMD(0x10,	NO_MELTDOWN | NO_SSB | NO_L1TF | NO_MDS | NO_SWAPGS | NO_ITLB_MULTIHIT | NO_MMIO | NO_BHI),
	VULNWL_AMD(0x11,	NO_MELTDOWN | NO_SSB | NO_L1TF | NO_MDS | NO_SWAPGS | NO_ITLB_MULTIHIT | NO_MMIO | NO_BHI),
	VULNWL_AMD(0x12,	NO_MELTDOWN | NO_SSB | NO_L1TF | NO_MDS | NO_SWAPGS | NO_ITLB_MULTIHIT | NO_MMIO | NO_BHI),
// snipped
};
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/common.c#L1116](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/common.c#L1116)
```c
#define NO_SSB			BIT(2)
```

### `ARCH_CAP_SSB_NO` / `X86_FEATURE_AMD_SSB_NO`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/msr-index.h#L116](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/msr-index.h#L116)
```c
#define MSR_IA32_ARCH_CAPABILITIES	0x0000010a
// snipped
#define ARCH_CAP_SSB_NO			BIT(4)	/*
						 * Not susceptible to Speculative Store Bypass
						 * attack, so no Speculative Store Bypass
						 * control required.
						 */
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L347](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L347)
```c
/* AMD-defined CPU features, CPUID level 0x80000008 (EBX), word 13 */
// snipped
#define X86_FEATURE_AMD_SSB_NO		(13*32+26) /* Speculative Store Bypass is fixed in hardware. */
```

## Mitigation Detection

### Summary

- SSBD (Speculative Store Bypass Disable) is a feature to disable speculation.
- It is enumerated on:
    - Intel: CPUID.(EAX=7h,ECX=0):EDX.SSBD[bit 31]
    - AMD: CPUID.80000008:EBX.SSBD[bit 24] / CPUID.80000008:EBX.VIRT_SSBD[bit 25] for virtualized guests
        - Some AMD families a different mechanism using LS_CFG MSR (address 0xC0011020) and the bit that should be used depends on the family.

### `init_speculation_control()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/common.c#L931](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/common.c#L931)
```c
static void init_speculation_control(struct cpuinfo_x86 *c)
{
// snipped
	if (cpu_has(c, X86_FEATURE_SPEC_CTRL_SSBD) ||
	    cpu_has(c, X86_FEATURE_VIRT_SSBD))
		set_cpu_cap(c, X86_FEATURE_SSBD);
// snipped
	if (cpu_has(c, X86_FEATURE_AMD_SSBD)) {
		set_cpu_cap(c, X86_FEATURE_SSBD);
		set_cpu_cap(c, X86_FEATURE_MSR_SPEC_CTRL);
		clear_cpu_cap(c, X86_FEATURE_VIRT_SSBD);
	}
}
```

### `X86_FEATURE_SPEC_CTRL_SSBD` / `X86_FEATURE_VIRT_SSBD` / `X86_FEATURE_AMD_SSBD`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L440](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L440)
```c
/* Intel-defined CPU features, CPUID level 0x00000007:0 (EDX), word 18 */
// snipped
#define X86_FEATURE_SPEC_CTRL_SSBD	(18*32+31) /* Speculative Store Bypass Disable */
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L346](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L346)
```c
/* AMD-defined CPU features, CPUID level 0x80000008 (EBX), word 13 */
// snipped
#define X86_FEATURE_AMD_SSBD		(13*32+24) /* Speculative Store Bypass Disable */
#define X86_FEATURE_VIRT_SSBD		(13*32+25) /* "virt_ssbd" Virtualized Speculative Store Bypass Disable */
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L345-L346](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L345-L346)
```c
/* AMD-defined CPU features, CPUID level 0x80000008 (EBX), word 13 */
#define X86_FEATURE_CLZERO		(13*32+ 0) /* "clzero" CLZERO instruction */
#define X86_FEATURE_IRPERF		(13*32+ 1) /* "irperf" Instructions Retired Count */
#define X86_FEATURE_XSAVEERPTR		(13*32+ 2) /* "xsaveerptr" Always save/restore FP error pointers */
#define X86_FEATURE_RDPRU		(13*32+ 4) /* "rdpru" Read processor register at user level */
#define X86_FEATURE_WBNOINVD		(13*32+ 9) /* "wbnoinvd" WBNOINVD instruction */
#define X86_FEATURE_AMD_IBPB		(13*32+12) /* Indirect Branch Prediction Barrier */
#define X86_FEATURE_AMD_IBRS		(13*32+14) /* Indirect Branch Restricted Speculation */
#define X86_FEATURE_AMD_STIBP		(13*32+15) /* Single Thread Indirect Branch Predictors */
#define X86_FEATURE_AMD_STIBP_ALWAYS_ON	(13*32+17) /* Single Thread Indirect Branch Predictors always-on preferred */
#define X86_FEATURE_AMD_PPIN		(13*32+23) /* "amd_ppin" Protected Processor Inventory Number */
#define X86_FEATURE_AMD_SSBD		(13*32+24) /* Speculative Store Bypass Disable */
#define X86_FEATURE_VIRT_SSBD		(13*32+25) /* "virt_ssbd" Virtualized Speculative Store Bypass Disable */
```

### `X86_FEATURE_LS_CFG_SSBD`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L506](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L506)
```c
#define X86_FEATURE_LS_CFG_SSBD		( 7*32+24)  /* AMD SSBD implementation via LS_CFG MSR */
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/amd.c#L419](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/amd.c#L419)
```c
static void bsp_init_amd(struct cpuinfo_x86 *c)
{
// snipped
	if (!boot_cpu_has(X86_FEATURE_AMD_SSBD) &&
	    !boot_cpu_has(X86_FEATURE_VIRT_SSBD) &&
	    c->x86 >= 0x15 && c->x86 <= 0x17) {
		unsigned int bit;

		switch (c->x86) {
		case 0x15: bit = 54; break;
		case 0x16: bit = 33; break;
		case 0x17: bit = 10; break;
		default: return;
		}
		/*
		 * Try to cache the base value so further operations can
		 * avoid RMW. If that faults, do not enable SSBD.
		 */
		if (!rdmsrl_safe(MSR_AMD64_LS_CFG, &x86_amd_ls_cfg_base)) {
			setup_force_cpu_cap(X86_FEATURE_LS_CFG_SSBD);
			setup_force_cpu_cap(X86_FEATURE_SSBD);
			x86_amd_ls_cfg_ssbd_mask = 1ULL << bit;
		}
	}
// snipped
}
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/msr-index.h#L594](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/msr-index.h#L594)
```c
#define MSR_AMD64_LS_CFG		0xc0011020
```

## Mitigation Selection

- Kernel parameter `spec_store_bypass_disable=` can take one of the following five values:
    - "on": Unconditionally disable speculative store bypass
    - "off": Unconditionally permits speculative store bypass
    - "prctl": Allows userspace to disable speculative store bypass via `prctl()`
    - "seccomp": All seccomp threads disable speculative store bypass in addition to `prctl()`.
    - "auto" (default): The kernel detects whether the processor is affected and which mitigation to apply.
- Mitigation is selected as follows:
    - If SSBD is not supported => `SPEC_STORE_BYPASS_NONE`
    - If the processor is not affected and `spec_store_bypass_disable=off` or `spec_store_bypass_disable=auto` => `SPEC_STORE_BYPASS_NONE`
    - If `spec_store_bypass_disable=seccomp` and `CONFIG_SECCOMP` is enabled => `SPEC_STORE_BYPASS_SECCOMP`
    - If `spec_store_bypass_disable=seccomp` and `CONFIG_SECCOMP` is not enabled => `SPEC_STORE_BYPASS_PRCTL`
    - If `spec_store_bypass_disable=on` => `SPEC_STORE_BYPASS_DISABLE`
    - If `spec_store_bypass_disable=auto` or `spec_store_bypass_disable=prctl` => `SPEC_STORE_BYPASS_PRCTL`
    - If `spec_store_bypass_disable=off` => `SPEC_STORE_BYPASS_NONE`
- Mitigation application
    - Intel: `IA32_SPEC_CTRL.SSBD[bit 2] = 1`
    - AMD: `IA32_SPEC_CTRL.SSBD[bit 2] = 1`
        - Some AMD families use LS_CFG MSR (address 0xC0011020) and the bit that should be used depends on the family.
        - Some virtualized AMD guests use VIRT_SPEC_CTRL MSR (address 0xC001011F).

### `ssb_mitigation_options[]`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2028](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2028)
```c
static const struct {
	const char *option;
	enum ssb_mitigation_cmd cmd;
} ssb_mitigation_options[]  __initconst = {
	{ "auto",	SPEC_STORE_BYPASS_CMD_AUTO },    /* Platform decides */
	{ "on",		SPEC_STORE_BYPASS_CMD_ON },      /* Disable Speculative Store Bypass */
	{ "off",	SPEC_STORE_BYPASS_CMD_NONE },    /* Don't touch Speculative Store Bypass */
	{ "prctl",	SPEC_STORE_BYPASS_CMD_PRCTL },   /* Disable Speculative Store Bypass via prctl */
	{ "seccomp",	SPEC_STORE_BYPASS_CMD_SECCOMP }, /* Disable Speculative Store Bypass via prctl and seccomp */
};
```

[https://elixir.bootlin.com/linux/v6.11.5/source/Documentation/admin-guide/kernel-parameters.txt#L6251-L6301](https://elixir.bootlin.com/linux/v6.11.5/source/Documentation/admin-guide/kernel-parameters.txt#L6251-L6301)
```
	spec_store_bypass_disable=
			[HW,EARLY] Control Speculative Store Bypass (SSB) Disable mitigation
			(Speculative Store Bypass vulnerability)

			Certain CPUs are vulnerable to an exploit against a
			a common industry wide performance optimization known
			as "Speculative Store Bypass" in which recent stores
			to the same memory location may not be observed by
			later loads during speculative execution. The idea
			is that such stores are unlikely and that they can
			be detected prior to instruction retirement at the
			end of a particular speculation execution window.

			In vulnerable processors, the speculatively forwarded
			store can be used in a cache side channel attack, for
			example to read memory to which the attacker does not
			directly have access (e.g. inside sandboxed code).

			This parameter controls whether the Speculative Store
			Bypass optimization is used.

			On x86 the options are:

			on      - Unconditionally disable Speculative Store Bypass
			off     - Unconditionally enable Speculative Store Bypass
			auto    - Kernel detects whether the CPU model contains an
				  implementation of Speculative Store Bypass and
				  picks the most appropriate mitigation. If the
				  CPU is not vulnerable, "off" is selected. If the
				  CPU is vulnerable the default mitigation is
				  architecture and Kconfig dependent. See below.
			prctl   - Control Speculative Store Bypass per thread
				  via prctl. Speculative Store Bypass is enabled
				  for a process by default. The state of the control
				  is inherited on fork.
			seccomp - Same as "prctl" above, but all seccomp threads
				  will disable SSB unless they explicitly opt out.

			Default mitigations:
			X86:	"prctl"

			On powerpc the options are:

			on,auto - On Power8 and Power9 insert a store-forwarding
				  barrier on kernel entry and exit. On Power7
				  perform a software flush on kernel entry and
				  exit.
			off	- No action.

			Not specifying this option is equivalent to
			spec_store_bypass_disable=auto.
```

### `ssb_parse_cmdline()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2039](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2039)
```c
static enum ssb_mitigation_cmd __init ssb_parse_cmdline(void)
{
	enum ssb_mitigation_cmd cmd = SPEC_STORE_BYPASS_CMD_AUTO;
	char arg[20];
	int ret, i;

	if (cmdline_find_option_bool(boot_command_line, "nospec_store_bypass_disable") ||
	    cpu_mitigations_off()) {
		return SPEC_STORE_BYPASS_CMD_NONE;
	} else {
		ret = cmdline_find_option(boot_command_line, "spec_store_bypass_disable",
					  arg, sizeof(arg));
		if (ret < 0)
			return SPEC_STORE_BYPASS_CMD_AUTO;

		for (i = 0; i < ARRAY_SIZE(ssb_mitigation_options); i++) {
			if (!match_option(arg, ret, ssb_mitigation_options[i].option))
				continue;

			cmd = ssb_mitigation_options[i].cmd;
			break;
		}

		if (i >= ARRAY_SIZE(ssb_mitigation_options)) {
			pr_err("unknown option (%s). Switching to AUTO select\n", arg);
			return SPEC_STORE_BYPASS_CMD_AUTO;
		}
	}

	return cmd;
}
```

### `ssb_select_mitigation()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2131](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2131)
```c
static void ssb_select_mitigation(void)
{
	ssb_mode = __ssb_select_mitigation();

	if (boot_cpu_has_bug(X86_BUG_SPEC_STORE_BYPASS))
		pr_info("%s\n", ssb_strings[ssb_mode]);
}
```

### `enum ssb_mitigation`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/nospec-branch.h#L513](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/nospec-branch.h#L513)
```c
/* The Speculative Store Bypass disable variants */
enum ssb_mitigation {
	SPEC_STORE_BYPASS_NONE,
	SPEC_STORE_BYPASS_DISABLE,
	SPEC_STORE_BYPASS_PRCTL,
	SPEC_STORE_BYPASS_SECCOMP,
};
```

### `__ssb_select_mitigation()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2071](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2071)
```c
static enum ssb_mitigation __init __ssb_select_mitigation(void)
{
	enum ssb_mitigation mode = SPEC_STORE_BYPASS_NONE;
	enum ssb_mitigation_cmd cmd;

	if (!boot_cpu_has(X86_FEATURE_SSBD))
		return mode;

	cmd = ssb_parse_cmdline();
	if (!boot_cpu_has_bug(X86_BUG_SPEC_STORE_BYPASS) &&
	    (cmd == SPEC_STORE_BYPASS_CMD_NONE ||
	     cmd == SPEC_STORE_BYPASS_CMD_AUTO))
		return mode;

	switch (cmd) {
	case SPEC_STORE_BYPASS_CMD_SECCOMP:
		/*
		 * Choose prctl+seccomp as the default mode if seccomp is
		 * enabled.
		 */
		if (IS_ENABLED(CONFIG_SECCOMP))
			mode = SPEC_STORE_BYPASS_SECCOMP;
		else
			mode = SPEC_STORE_BYPASS_PRCTL;
		break;
	case SPEC_STORE_BYPASS_CMD_ON:
		mode = SPEC_STORE_BYPASS_DISABLE;
		break;
	case SPEC_STORE_BYPASS_CMD_AUTO:
	case SPEC_STORE_BYPASS_CMD_PRCTL:
		mode = SPEC_STORE_BYPASS_PRCTL;
		break;
	case SPEC_STORE_BYPASS_CMD_NONE:
		break;
	}

	/*
	 * We have three CPU feature flags that are in play here:
	 *  - X86_BUG_SPEC_STORE_BYPASS - CPU is susceptible.
	 *  - X86_FEATURE_SSBD - CPU is able to turn off speculative store bypass
	 *  - X86_FEATURE_SPEC_STORE_BYPASS_DISABLE - engage the mitigation
	 */
	if (mode == SPEC_STORE_BYPASS_DISABLE) {
		setup_force_cpu_cap(X86_FEATURE_SPEC_STORE_BYPASS_DISABLE);
		/*
		 * Intel uses the SPEC CTRL MSR Bit(2) for this, while AMD may
		 * use a completely different MSR and bit dependent on family.
		 */
		if (!static_cpu_has(X86_FEATURE_SPEC_CTRL_SSBD) &&
		    !static_cpu_has(X86_FEATURE_AMD_SSBD)) {
			x86_amd_ssb_disable();
		} else {
			x86_spec_ctrl_base |= SPEC_CTRL_SSBD;
			update_spec_ctrl(x86_spec_ctrl_base);
		}
	}

	return mode;
}
```

It parses the kernel command line first via `ssb_parse_cmdline()`

### `x86_amd_ssb_disable()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L222](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L222)
```c
static void x86_amd_ssb_disable(void)
{
	u64 msrval = x86_amd_ls_cfg_base | x86_amd_ls_cfg_ssbd_mask;

	if (boot_cpu_has(X86_FEATURE_VIRT_SSBD))
		wrmsrl(MSR_AMD64_VIRT_SPEC_CTRL, SPEC_CTRL_SSBD);
	else if (boot_cpu_has(X86_FEATURE_LS_CFG_SSBD))
		wrmsrl(MSR_AMD64_LS_CFG, msrval);
}
```

## Speculation Control via prctl()

### Summary

- Speculative store bypass can be disabled via prctl():
    - `PR_SPEC_ENABLE` (1): Enable speculation and disable mitigation.
    - `PR_SPEC_DISABLE` (2): Disable speculation and enable mitigation.
    - `PR_SPEC_FORCE_DISABLE` (3): Same as `PR_SPEC_DISABLE` but cannot be undone. A subsequent `prctl(..., PR_SPEC_ENABLE)` will fail.
    - `PR_SPEC_DISABLE_NOEXEC` (4): Same as `PR_SPEC_DISABLE` but the state will be cleared on `execve()`.

### `prctl()`

[https://elixir.bootlin.com/linux/v6.11.5/source/kernel/sys.c#L2673](https://elixir.bootlin.com/linux/v6.11.5/source/kernel/sys.c#L2673)
```c
SYSCALL_DEFINE5(prctl, int, option, unsigned long, arg2, unsigned long, arg3,
		unsigned long, arg4, unsigned long, arg5)
{
// snipped
	switch (option) {
// snipped
	case PR_SET_SPECULATION_CTRL:
		if (arg4 || arg5)
			return -EINVAL;
		error = arch_prctl_spec_ctrl_set(me, arg2, arg3);
		break;
// snipped
	}
	return error;
}
```

### `arch_prctl_spec_ctrl_set()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2285](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2285)
```c
int arch_prctl_spec_ctrl_set(struct task_struct *task, unsigned long which,
			     unsigned long ctrl)
{
	switch (which) {
	case PR_SPEC_STORE_BYPASS:
		return ssb_prctl_set(task, ctrl);
// snipped
	}
}
```

### `ssb_prctl_set()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2177](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2177)
```c
static int ssb_prctl_set(struct task_struct *task, unsigned long ctrl)
{
	if (ssb_mode != SPEC_STORE_BYPASS_PRCTL &&
	    ssb_mode != SPEC_STORE_BYPASS_SECCOMP)
		return -ENXIO;

	switch (ctrl) {
	case PR_SPEC_ENABLE:
		/* If speculation is force disabled, enable is not allowed */
		if (task_spec_ssb_force_disable(task))
			return -EPERM;
		task_clear_spec_ssb_disable(task);
		task_clear_spec_ssb_noexec(task);
		task_update_spec_tif(task);
		break;
	case PR_SPEC_DISABLE:
		task_set_spec_ssb_disable(task);
		task_clear_spec_ssb_noexec(task);
		task_update_spec_tif(task);
		break;
	case PR_SPEC_FORCE_DISABLE:
		task_set_spec_ssb_disable(task);
		task_set_spec_ssb_force_disable(task);
		task_clear_spec_ssb_noexec(task);
		task_update_spec_tif(task);
		break;
	case PR_SPEC_DISABLE_NOEXEC:
		if (task_spec_ssb_force_disable(task))
			return -EPERM;
		task_set_spec_ssb_disable(task);
		task_set_spec_ssb_noexec(task);
		task_update_spec_tif(task);
		break;
	default:
		return -ERANGE;
	}
	return 0;
}
```

[https://elixir.bootlin.com/linux/v6.11.5/source/include/uapi/linux/prctl.h#L217](https://elixir.bootlin.com/linux/v6.11.5/source/include/uapi/linux/prctl.h#L217)
```c
/* Return and control values for PR_SET/GET_SPECULATION_CTRL */
# define PR_SPEC_NOT_AFFECTED		0
# define PR_SPEC_PRCTL			(1UL << 0)
# define PR_SPEC_ENABLE			(1UL << 1)
# define PR_SPEC_DISABLE		(1UL << 2)
# define PR_SPEC_FORCE_DISABLE		(1UL << 3)
# define PR_SPEC_DISABLE_NOEXEC		(1UL << 4)
```

[https://docs.kernel.org/userspace-api/spec_ctrl.html](https://docs.kernel.org/userspace-api/spec_ctrl.html)

- `PR_SPEC_ENABLE` (1): Enable speculation and disable mitigation.
- `PR_SPEC_DISABLE` (2): Disable speculation and enable mitigation.
- `PR_SPEC_FORCE_DISABLE` (3): Same as `PR_SPEC_DISABLE` but cannot be undone. A subsequent `prctl(..., PR_SPEC_ENABLE)` will fail.
- `PR_SPEC_DISABLE_NOEXEC` (4): Same as `PR_SPEC_DISABLE` but the state will be cleared on `execve()`.

### `task_{,set_,clear_}_{spec_ssb_disable,spec_ssb_noexec,spec_ssb_force_disable}()`

[https://elixir.bootlin.com/linux/v6.11.4/source/include/linux/sched.h#L1744](https://elixir.bootlin.com/linux/v6.11.4/source/include/linux/sched.h#L1744)
```c
#define TASK_PFA_TEST(name, func)					\
	static inline bool task_##func(struct task_struct *p)		\
	{ return test_bit(PFA_##name, &p->atomic_flags); }

#define TASK_PFA_SET(name, func)					\
	static inline void task_set_##func(struct task_struct *p)	\
	{ set_bit(PFA_##name, &p->atomic_flags); }

#define TASK_PFA_CLEAR(name, func)					\
	static inline void task_clear_##func(struct task_struct *p)	\
	{ clear_bit(PFA_##name, &p->atomic_flags); }

// snipped
TASK_PFA_TEST(SPEC_SSB_DISABLE, spec_ssb_disable)
TASK_PFA_SET(SPEC_SSB_DISABLE, spec_ssb_disable)
TASK_PFA_CLEAR(SPEC_SSB_DISABLE, spec_ssb_disable)

TASK_PFA_TEST(SPEC_SSB_NOEXEC, spec_ssb_noexec)
TASK_PFA_SET(SPEC_SSB_NOEXEC, spec_ssb_noexec)
TASK_PFA_CLEAR(SPEC_SSB_NOEXEC, spec_ssb_noexec)

TASK_PFA_TEST(SPEC_SSB_FORCE_DISABLE, spec_ssb_force_disable)
TASK_PFA_SET(SPEC_SSB_FORCE_DISABLE, spec_ssb_force_disable)
```

### `task_update_spec_tif()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2142](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2142)
```c
static void task_update_spec_tif(struct task_struct *tsk)
{
	/* Force the update of the real TIF bits */
	set_tsk_thread_flag(tsk, TIF_SPEC_FORCE_UPDATE);

	/*
	 * Immediately update the speculation control MSRs for the current
	 * task, but for a non-current task delay setting the CPU
	 * mitigation until it is scheduled next.
	 *
	 * This can only happen for SECCOMP mitigation. For PRCTL it's
	 * always the current task.
	 */
	if (tsk == current)
		speculation_ctrl_update_current();
}
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/thread_info.h#L104](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/thread_info.h#L104)
```c
#define TIF_SPEC_FORCE_UPDATE	23	/* Force speculation MSR update in context switch */
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/process.c#L674](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/process.c#L674)
```c
/* Called from seccomp/prctl update */
void speculation_ctrl_update_current(void)
{
	preempt_disable();
	speculation_ctrl_update(speculation_ctrl_update_tif(current));
	preempt_enable();
}
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/process.c#L646](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/process.c#L646)
```c
static unsigned long speculation_ctrl_update_tif(struct task_struct *tsk)
{
	if (test_and_clear_tsk_thread_flag(tsk, TIF_SPEC_FORCE_UPDATE)) {
		if (task_spec_ssb_disable(tsk))
			set_tsk_thread_flag(tsk, TIF_SSBD);
		else
			clear_tsk_thread_flag(tsk, TIF_SSBD);

// snipped
	}
	/* Return the updated threadinfo flags*/
	return read_task_thread_flags(tsk);
}
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/process.c#L663](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/process.c#L663)
```c
void speculation_ctrl_update(unsigned long tif)
{
	unsigned long flags;

	/* Forced update. Make sure all relevant TIF flags are different */
	local_irq_save(flags);
	__speculation_ctrl_update(~tif, tif);
	local_irq_restore(flags);
}
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/process.c#L613](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/process.c#L613)
```c
/*
 * Update the MSRs managing speculation control, during context switch.
 *
 * tifp: Previous task's thread flags
 * tifn: Next task's thread flags
 */
static __always_inline void __speculation_ctrl_update(unsigned long tifp,
						      unsigned long tifn)
{
	unsigned long tif_diff = tifp ^ tifn;
	u64 msr = x86_spec_ctrl_base;
	bool updmsr = false;

	lockdep_assert_irqs_disabled();

	/* Handle change of TIF_SSBD depending on the mitigation method. */
	if (static_cpu_has(X86_FEATURE_VIRT_SSBD)) {
		if (tif_diff & _TIF_SSBD)
			amd_set_ssb_virt_state(tifn);
	} else if (static_cpu_has(X86_FEATURE_LS_CFG_SSBD)) {
		if (tif_diff & _TIF_SSBD)
			amd_set_core_ssb_state(tifn);
	} else if (static_cpu_has(X86_FEATURE_SPEC_CTRL_SSBD) ||
		   static_cpu_has(X86_FEATURE_AMD_SSBD)) {
		updmsr |= !!(tif_diff & _TIF_SSBD);
		msr |= ssbd_tif_to_spec_ctrl(tifn);
	}

	/* Only evaluate TIF_SPEC_IB if conditional STIBP is enabled. */
	if (IS_ENABLED(CONFIG_SMP) &&
	    static_branch_unlikely(&switch_to_cond_stibp)) {
		updmsr |= !!(tif_diff & _TIF_SPEC_IB);
		msr |= stibp_tif_to_spec_ctrl(tifn);
	}

	if (updmsr)
		update_spec_ctrl_cond(msr);
}
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L81](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L81)
```c
/*
 * Keep track of the SPEC_CTRL MSR value for the current task, which may differ
 * from x86_spec_ctrl_base due to STIBP/SSB in __speculation_ctrl_update().
 */
void update_spec_ctrl_cond(u64 val)
{
	if (this_cpu_read(x86_spec_ctrl_current) == val)
		return;

	this_cpu_write(x86_spec_ctrl_current, val);

	/*
	 * When KERNEL_IBRS this MSR is written on return-to-user, unless
	 * forced the update can be delayed until that time.
	 */
	if (!cpu_feature_enabled(X86_FEATURE_KERNEL_IBRS))
		wrmsrl(MSR_IA32_SPEC_CTRL, val);
}
```

## Speculation Control via seccomp

### Summary

- If `spec_store_bypass_disable=seccomp`, seccomp applies the same effect of `prctl()` with `PR_SPEC_FORCE_DISABLE`.

### `seccomp_assign_mode()`

[https://elixir.bootlin.com/linux/v6.11.5/source/kernel/seccomp.c#L449](https://elixir.bootlin.com/linux/v6.11.5/source/kernel/seccomp.c#L449)
```c
static inline void seccomp_assign_mode(struct task_struct *task,
				       unsigned long seccomp_mode,
				       unsigned long flags)
{
// snipped
	/* Assume default seccomp processes want spec flaw mitigation. */
	if ((flags & SECCOMP_FILTER_FLAG_SPEC_ALLOW) == 0)
		arch_seccomp_spec_mitigate(task);
// snipped
}
```

[https://elixir.bootlin.com/linux/v6.11.5/source/include/uapi/linux/seccomp.h#L23](https://elixir.bootlin.com/linux/v6.11.5/source/include/uapi/linux/seccomp.h#L23)
```c
/* Valid flags for SECCOMP_SET_MODE_FILTER */
// snipped
#define SECCOMP_FILTER_FLAG_SPEC_ALLOW		(1UL << 2)
```

### `arch_seccomp_spec_mitigate()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2296](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2296)
```c
#ifdef CONFIG_SECCOMP
void arch_seccomp_spec_mitigate(struct task_struct *task)
{
	if (ssb_mode == SPEC_STORE_BYPASS_SECCOMP)
		ssb_prctl_set(task, PR_SPEC_FORCE_DISABLE);
// snipped
}
#endif
```


# History of Kernel Patches

Note: The lines marked * are not much related to SSB. 

| Date       | No.     | Patch | Links |
|------------|---------|-------|-------|
| 2018-03-21 | -       | _The final SSB bundle_ | [[lore]](https://lore.kernel.org/all/alpine.DEB.2.21.1805211542220.1599@nanos.tec.linutronix.de/) [[commit]](https://github.com/torvalds/linux/commit/3b78ce4a34b761c7fe13520de822984019ff1a8f) |
| 2018-05-03 | 01      | (*) x86/nospec: Simplify alternative_msr_write() | [[commit]](https://github.com/torvalds/linux/commit/1aa7a5735a41418d8e01fa7c9565eb2657e2ea3f) |
| 2018-05-03 | 02      | (*) x86/bugs: Concentrate bug detection into a separate function | [[commit]](https://github.com/torvalds/linux/commit/4a28bfe3267b68e22c663ac26185aa16c9b879ef) |
| 2018-05-03 | 03      | (*) x86/bugs: Concentrate bug reporting into a separate function | [[commit]](https://github.com/torvalds/linux/commit/d1059518b4789cabe34bb4b714d07e6089c82ca1) |
| 2018-05-03 | 04      | (*) x86/bugs: Read SPEC_CTRL MSR during boot and re-use reserved bits | [[commit]](https://github.com/torvalds/linux/commit/1b86883ccb8d5d9506529d42dbe1a5257cb30b18) |
| 2018-05-03 | 05      | (*) x86/bugs, KVM: Support the combination of guest and host IBRS | [[commit]](https://github.com/torvalds/linux/commit/5cf687548705412da47c9cec342fd952d71ed3d5) |
| 2018-05-03 | 06      | x86/bugs: Expose /sys/../spec_store_bypass | [[commit]](https://github.com/torvalds/linux/commit/c456442cd3a59eeb1d60293c26cbe2ff2c4e42cf) |
| 2018-05-03 | 07      | x86/cpufeatures: Add X86_FEATURE_RDS | [[commit]](https://github.com/torvalds/linux/commit/0cc5fa00b0a88dad140b4e5c2cead9951ad36822) |
| 2018-05-03 | 08      | x86/bugs: Provide boot parameters for the spec_store_bypass_disable mitigation | [[commit]](https://github.com/torvalds/linux/commit/24f7fc83b9204d20f878c57cb77d261ae825e033) |
| 2018-05-03 | 09      | x86/bugs/intel: Set proper CPU features and setup RDS | [[commit]](https://github.com/torvalds/linux/commit/772439717dbf703b39990be58d8d4e3e4ad0598a) |
| 2018-05-03 | 10      | x86/bugs: Whitelist allowed SPEC_CTRL MSR values | [[commit]](https://github.com/torvalds/linux/commit/1115a859f33276fe8afb31c60cf9d8e657872558) |
| 2018-05-03 | 11      | x86/bugs/AMD: Add support to disable RDS on Fam[15,16,17]h if requested | [[commit]](https://github.com/torvalds/linux/commit/764f3c21588a059cd783c6ba0734d4db2d72822d) |
| 2018-05-03 | 12      | x86/KVM/VMX: Expose SPEC_CTRL Bit(2) to the guest | [[commit]](https://github.com/torvalds/linux/commit/da39556f66f5cfe8f9c989206974f1cb16ca5d7c) |
| 2018-05-03 | 13      | x86/speculation: Create spec-ctrl.h to avoid include hell | [[commit]](https://github.com/torvalds/linux/commit/28a2775217b17208811fa43a9e96bd1fdf417b86) |
| 2018-05-03 | 14      | prctl: Add speculation control prctls | [[commit]](https://github.com/torvalds/linux/commit/b617cfc858161140d69cc0b5cc211996b557a1c7) |
| 2018-05-03 | 15      | x86/process: Allow runtime control of Speculative Store Bypass | [[commit]](https://github.com/torvalds/linux/commit/885f82bfbc6fefb6664ea27965c3ab9ac4194b8c) |
| 2018-05-03 | 16      | x86/speculation: Add prctl for Speculative Store Bypass mitigation | [[commit]](https://github.com/torvalds/linux/commit/a73ec77ee17ec556fe7f165d00314cb7c047b1ac) |
| 2018-05-03 | 17      | nospec: Allow getting/setting on non-current task | [[commit]](https://github.com/torvalds/linux/commit/7bbf1373e228840bb0295a2ca26d548ef37f448e) |
| 2018-05-03 | 18      | proc: Provide details on speculation flaw mitigations | [[commit]](https://github.com/torvalds/linux/commit/fae1fa0fc6cca8beee3ab8ed71d54f9a78fa3f64) |
| 2018-05-03 | 19      | seccomp: Enable speculation flaw mitigations | [[commit]](https://github.com/torvalds/linux/commit/5c3070890d06ff82eecb808d02d2ca39169533ef) |
| 2018-05-04 | 20      | x86/bugs: Make boot modes __ro_after_init | [[commit]](https://github.com/torvalds/linux/commit/f9544b2b076ca90d887c5ae5d74fab4c21bb7c13) |
| 2018-05-04 | 21      | prctl: Add force disable speculation | [[commit]](https://github.com/torvalds/linux/commit/356e4bfff2c5489e016fdb925adbf12a1e3950ee) |
| 2018-05-04 | 22      | seccomp: Use PR_SPEC_FORCE_DISABLE | [[commit]](https://github.com/torvalds/linux/commit/b849a812f7eb92e96d1c8239b06581b2cfd8b275) |
| 2018-05-04 | 23      | seccomp: Add filter flag to opt-out of SSB mitigation | [[commit]](https://github.com/torvalds/linux/commit/00a02d0c502a06d15e07b857f8ff921e3e402675) |
| 2018-05-04 | 24      | seccomp: Move speculation migitation control to arch code | [[commit]](https://github.com/torvalds/linux/commit/8bf37d8c067bb7eb8e7c381bdadf9bd89182b6bc) |
| 2018-05-04 | 25      | x86/speculation: Make "seccomp" the default mode for Speculative Store Bypass | [[commit]](https://github.com/torvalds/linux/commit/f21b53b20c754021935ea43364dbf53778eeba32) |
| 2018-05-09 | 26      | x86/bugs: Rename _RDS to _SSBD | [[commit]](https://github.com/torvalds/linux/commit/9f65fb29374ee37856dbad847b4e121aab72b510) |
| 2018-05-09 | 27      | proc: Use underscores for SSBD in status | [[commit]](https://github.com/torvalds/linux/commit/e96f46ee8587607a828f783daa6eb5b44d25004d) |
| 2018-05-09 | 28      | Documentation/spec_ctrl: Do some minor cleanups | [[commit]](https://github.com/torvalds/linux/commit/dd0792699c4058e63c0715d9a7c2d40226fcdddc) |
| 2018-05-10 | 29      | x86/bugs: Fix __ssb_select_mitigation() return type | [[commit]](https://github.com/torvalds/linux/commit/d66d8ff3d21667b41eddbe86b35ab411e40d8c5f) |
| 2018-05-10 | 30      | (*) x86/bugs: Make cpu_show_common() static | [[commit]](https://github.com/torvalds/linux/commit/7bb4d366cba992904bffa4820d24e70a3de93e76) |
| 2018-05-12 | 31      | (*) x86/bugs: Fix the parameters alignment and missing void | [[commit]](https://github.com/torvalds/linux/commit/ffed645e3be0e32f8e9ab068d257aee8d0fe8eec) |
| 2018-05-13 | 32      | (*) x86/cpu: Make alternative_msr_write work for 32-bit code | [[commit]](https://github.com/torvalds/linux/commit/5f2b745f5e1304f438f9b2cd03ebc8120b6e0d3b) |
| 2018-05-17 | 33      | (*) KVM: SVM: Move spec control call after restore of GS | [[commit]](https://github.com/torvalds/linux/commit/15e6c22fd8e5a42c5ed6d487b7c9fe44c2517765) |
| 2018-05-17 | 34      | (*) x86/speculation: Use synthetic bits for IBRS/IBPB/STIBP | [[commit]](https://github.com/torvalds/linux/commit/e7c587da125291db39ddf1f49b18e5970adbac17) |
| 2018-05-17 | 35      | (*) x86/cpufeatures: Disentangle MSR_SPEC_CTRL enumeration from IBRS | [[commit]](https://github.com/torvalds/linux/commit/7eb8956a7fec3c1f0abc2a5517dada99ccc8a961) |
| 2018-05-17 | 36      | x86/cpufeatures: Disentangle SSBD enumeration | [[commit]](https://github.com/torvalds/linux/commit/52817587e706686fcdb27f14c1b000c92f266c96) |
| 2018-05-17 | 37      | x86/cpufeatures: Add FEATURE_ZEN | [[commit]](https://github.com/torvalds/linux/commit/d1035d971829dcf80e8686ccde26f94b0a069472) |
| 2018-05-17 | 38      | x86/speculation: Handle HT correctly on AMD | [[commit]](https://github.com/torvalds/linux/commit/1f50ddb4f4189243c05926b842dc1a0332195f31) |
| 2018-05-17 | 39      | x86/bugs, KVM: Extend speculation control for VIRT_SPEC_CTRL | [[commit]](https://github.com/torvalds/linux/commit/ccbcd2674472a978b48c91c1fbfb66c0ff959f24) |
| 2018-05-17 | 40      | x86/speculation: Add virtualized speculative store bypass disable support | [[commit]](https://github.com/torvalds/linux/commit/11fb0683493b2da112cd64c9dada221b52463bf7) |
| 2018-05-17 | 41      | x86/speculation: Rework speculative_store_bypass_update() | [[commit]](https://github.com/torvalds/linux/commit/0270be3e34efb05a88bc4c422572ece038ef3608) |
| 2018-05-17 | 42      | x86/bugs: Unify x86_spec_ctrl_{set_guest,restore_host} | [[commit]](https://github.com/torvalds/linux/commit/cc69b34989210f067b2c51d5539b5f96ebcc3a01) |
| 2018-05-17 | 43      | x86/bugs: Expose x86_spec_ctrl_base directly | [[commit]](https://github.com/torvalds/linux/commit/fa8ac4988249c38476f6ad678a4848a736373403) |
| 2018-05-17 | 44      | x86/bugs: Remove x86_spec_ctrl_set() | [[commit]](https://github.com/torvalds/linux/commit/4b59bdb569453a60b752b274ca61f009e37f4dae) |
| 2018-05-17 | 45      | x86/bugs: Rework spec_ctrl base and mask logic | [[commit]](https://github.com/torvalds/linux/commit/be6fcb5478e95bb1c91f489121238deb3abca46a) |
| 2018-05-17 | 46      | x86/speculation, KVM: Implement support for VIRT_SPEC_CTRL/LS_CFG | [[commit]](https://github.com/torvalds/linux/commit/47c61b3955cf712cadfc25635bf9bc174af030ea) |
| 2018-05-17 | 47      | KVM: SVM: Implement VIRT_SPEC_CTRL support for SSBD | [[commit]](https://github.com/torvalds/linux/commit/bc226f07dcd3c9ef0b7f6236fe356ea4a9cb4769) |
| 2018-05-18 | 48      | x86/bugs: Rename SSBD_NO to SSB_NO | [[commit]](https://github.com/torvalds/linux/commit/240da953fcc6a9008c92fae5b1f727ee5ed167ab) |
| 2018-05-19 | 49      | bpf: Prevent memory disambiguation attack | [[commit]](https://github.com/torvalds/linux/commit/af86ca4e3088fe5eacf2f7e58c01fa68ca067672) |
| 2018-06-01 | -       | _AMD SSB bits._ | [[lore]](https://lore.kernel.org/all/20180601145921.9500-2-konrad.wilk@oracle.com/T/#m769cb7b2fc4ce4ce942d00fab5fd4653ddb338e6) |
| 2018-06-06 | [1/3]   | x86/bugs: Add AMD's variant of SSB_NO | [[commit]](https://github.com/torvalds/linux/commit/24809860012e0130fbafe536709e08a22b3e959e) [[lore]](https://lore.kernel.org/all/20180601145921.9500-2-konrad.wilk@oracle.com/T/#m82c0d54dfc3cb607015c70a19121112936458280) |
| 2018-06-06 | [2/3]   | x86/bugs: Add AMD's SPEC_CTRL MSR usage | [[commit]](https://github.com/torvalds/linux/commit/6ac2f49edb1ef5446089c7c660017732886d62d6) [[lore]](https://lore.kernel.org/all/20180601145921.9500-2-konrad.wilk@oracle.com/T/#mb92323107f02ff29686178d1c88470e7f449ac03) |
| 2018-06-06 | [3/3]   | x86/bugs: Switch the selection of mitigation from CPU vendor to CPU features | [[commit]](https://github.com/torvalds/linux/commit/108fab4b5c8f12064ef86e02cb0459992affb30f) [[lore]](https://lore.kernel.org/all/20180601145921.9500-2-konrad.wilk@oracle.com/T/#ma7e8312b97580f8b7689a5e84840915db97d48a2) |
| 2021-10-04 | -       | x86: change default to spec_store_bypass_disable=prctl spectre_v2_user=prctl | [[commit]](https://github.com/torvalds/linux/commit/2f46993d83ff4abb310ef7b4beced56ba96f0d9d) [[lore]](https://lore.kernel.org/all/20201104235054.5678-1-aarcange@redhat.com/) |


# References

- General
    - [Speculative Store Bypass - Wikipedia](https://en.wikipedia.org/wiki/Speculative_Store_Bypass)
    - [NVD - cve-2018-3639](https://nvd.nist.gov/vuln/detail/cve-2018-3639)
- Google
    - [speculative execution, variant 4: speculative store bypass [42450580] - Project Zero](https://project-zero.issues.chromium.org/issues/42450580)
- Intel
    - [INTEL-SA-00115](https://www.intel.com/content/www/us/en/security-center/advisory/intel-sa-00115.html)
    - [Speculative Store Bypass / CVE-2018-3639 / INTEL-SA-00115](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/advisory-guidance/speculative-store-bypass.html)
    - [Speculative Execution Side Channel Mitigations](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/speculative-execution-side-channel-mitigations.html)
- AMD
    - [AMD Product Security](https://www.amd.com/en/resources/product-security.html#tabs-f457c20a21-item-06fa4e0a50-tab)
- Microsoft
    - [ADV180012 - Security Update Guide - Microsoft - Microsoft Guidance for Speculative Store Bypass](https://msrc.microsoft.com/update-guide/en-US/advisory/ADV180012)
    - [Analysis and mitigation of speculative store bypass (CVE-2018-3639) | MSRC Blog | Microsoft Security Response Center](https://msrc.microsoft.com/blog/2018/05/analysis-and-mitigation-of-speculative-store-bypass-cve-2018-3639/)
