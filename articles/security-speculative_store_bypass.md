---
title: 【Security】Spectulative Store Bypass
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

## x86-64

### Sysfs

- Access to `/sys/devices/system/cpu/vulnerabilities/spec_store_bypass` is handled by `cpu_show_spec_store_bypass()`.
- It returns one of the following four values depending on `ssb_mode` value:
    - `ssb_mode == SPEC_STORE_BYPASS_NONE` => "Vulnerable"
    - `ssb_mode == SPEC_STORE_BYPASS_DISABLE` => "Mitigation: Speculative Store Bypass disabled"
    - `ssb_mode == SPEC_STORE_BYPASS_PRCTL` => "Mitigation: Speculative Store Bypass disabled via prctl"
    - `ssb_mode == SPEC_STORE_BYPASS_SECCOMP` => "Mitigation: Speculative Store Bypass disabled via prctl and seccomp"

#### `cpu_show_spec_store_bypass()`

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

#### `cpu_show_common()`

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

#### `ssb_strings[]`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2021](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2021)
```c
static const char * const ssb_strings[] = {
	[SPEC_STORE_BYPASS_NONE]	= "Vulnerable",
	[SPEC_STORE_BYPASS_DISABLE]	= "Mitigation: Speculative Store Bypass disabled",
	[SPEC_STORE_BYPASS_PRCTL]	= "Mitigation: Speculative Store Bypass disabled via prctl",
	[SPEC_STORE_BYPASS_SECCOMP]	= "Mitigation: Speculative Store Bypass disabled via prctl and seccomp",
};
```

#### `enum ssb_mitigation`

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

### Vulnerability Detection

#### Summary

- The processor is considered vulnerable if all the following conditions are met:
    - It is not listed in the whitelist.
    - SSB_NO is not set
        - Intel: IA32_ARCH_CAPABILITIES.SSB_NO[bit 4]
        - AMD: CPUID.80000008h:EBX.SSB_NO[bit 26]

#### `X86_BUG_SPEC_STORE_BYPASS`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L506](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L506)
```c
#define X86_BUG_SPEC_STORE_BYPASS	X86_BUG(17) /* "spec_store_bypass" CPU is affected by speculative store bypass attack */
```

#### `cpu_set_bug_bits()`

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

#### `cpu_vuln_whitelist[]` / `NO_SSB`

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

#### `ARCH_CAP_SSB_NO` / `X86_FEATURE_AMD_SSB_NO`

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

### Mitigation Detection

#### Summary

- SSBD (Speculative Store Bypass Disable) is a feature to disable speculation.
- It is enumerated on:
    - Intel: CPUID.(EAX=7h,ECX=0):EDX.SSBD[bit 31]
    - AMD: CPUID.80000008:EBX.SSBD[bit 24] / CPUID.80000008:EBX.VIRT_SSBD[bit 25] for virtualized guests
        - Some AMD families a different mechanism using LS_CFG MSR (address 0xC0011020) and the bit that should be used depends on the family.

#### `init_speculation_control()`

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

#### `X86_FEATURE_SPEC_CTRL_SSBD` / `X86_FEATURE_VIRT_SSBD` / `X86_FEATURE_AMD_SSBD`

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

#### `X86_FEATURE_LS_CFG_SSBD`

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

### Mitigation Selection

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

#### `ssb_mitigation_options[]`

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

#### `ssb_parse_cmdline()`

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

#### `ssb_select_mitigation()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2131](https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L2131)
```c
static void ssb_select_mitigation(void)
{
	ssb_mode = __ssb_select_mitigation();

	if (boot_cpu_has_bug(X86_BUG_SPEC_STORE_BYPASS))
		pr_info("%s\n", ssb_strings[ssb_mode]);
}
```

#### `enum ssb_mitigation`

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

#### `__ssb_select_mitigation()`

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

#### `x86_amd_ssb_disable()`

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

### Speculation Control via prctl()

#### Summary

- Speculative store bypass can be disabled via prctl():
    - `PR_SPEC_ENABLE` (1): Enable speculation and disable mitigation.
    - `PR_SPEC_DISABLE` (2): Disable speculation and enable mitigation.
    - `PR_SPEC_FORCE_DISABLE` (3): Same as `PR_SPEC_DISABLE` but cannot be undone. A subsequent `prctl(..., PR_SPEC_ENABLE)` will fail.
    - `PR_SPEC_DISABLE_NOEXEC` (4): Same as `PR_SPEC_DISABLE` but the state will be cleared on `execve()`.

#### `prctl()`

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

#### `arch_prctl_spec_ctrl_set()`

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

#### `ssb_prctl_set()`

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

#### `task_{,set_,clear_}_{spec_ssb_disable,spec_ssb_noexec,spec_ssb_force_disable}()`

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

#### `task_update_spec_tif()`

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

### Speculation Control via seccomp

#### Summary

- If `spec_store_bypass_disable=seccomp`, seccomp applies the same effect of `prctl()` with `PR_SPEC_FORCE_DISABLE`.

#### `seccomp_assign_mode()`

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

#### `arch_seccomp_spec_mitigate()`

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


## arm64

### Summary

- Mitigation
    - Hardware-based mitigation (SSBS: Speculative Store Bypass Safe)
        - Discovery: `ID_AA64PFR1_EL1.SSBS >= 1`
        - Usage:
            - `PSTATE.SSBS`
                - `0`: Mitigated. Speculative store bypass is not permitted.
                - `1`: Not mitigated. Speculative store bypass is permitted.
            - `SCTLR_ELx.DSSBS`: The default value of PSTATE.SSBS on exception entry
                - `0`: Mitigated. Speculative store bypass is not permitted.
                - `1`: Not mitigated. Speculative store bypass is permitted.
    - Firmware-based mitigation (`SMCCC_ARCH_WORKAROUND_2`)
        - Discovery: via `SMCCC_ARCH_FEATURES`
        - Usage:
            - Call `SMCCC_ARCH_WORKAROUND_2` with
                - `0`: Mitigation enabled.
                - `1`: Mitigation disabled.
- Sysfs
    - Path: `/sys/devices/system/cpu/vulnerabilities/spec_store_bypass`
    - Reports one of the following three:
        - "Not affected" (`SPECTRE_UNAFFECTED`)
        - "Mitigation: Speculative Store Bypass disabled via prctl" (`SPECTRE_MITIGATED`)
        - "Vulnerable" (`SPECTRE_VULNERABLE`)
    - Determined as follows:
        - If the processor is listed in the safe list => `SPECTRE_UNAFFECTED`
        - If SSBS is supported => `SPECTRE_MITIGATED`
        - If `SMCCC_ARCH_WORKAROUND_2` is not required => `SPECTRE_UNAFFECTED`
        - If `SMCCC_ARCH_WORKAROUND_2` is supported => `SPECTRE_MITIGATED`
        - Else (i.e. not listed in the safe list and both SSBS and `SMCCC_ARCH_WORKAROUND_2` not supported) => `SPECTRE_VULNERABLE`
- Mitigation application
    - Kernel parameter `ssbd=` takes one of the following three:
        - `force-on`: Unconditionally enable mitigation for kernel and userspace.
        - `force-off`: Unconditionally disable mitigation for kernel and userspace.
        - `kernel` (aka dynamic mode): Always enable mitigation in the kernel and allow userspace to opt-in via prctl().
    - Kernel mitigation:
        - If SSBS is supported, `PSTATE.SSBS = 0` (to apply it immediately) and `SCTLR_ELx.DSSBS = 0` (to apply it on future kernel entries).
        - If `SMCCC_ARCH_WORKAROUND_2` is supported, calls `SMCCC_ARCH_WORKAROUND_2` with `1` (to apply it immediately).
            - In the dynamic mode, calls `SMCCC_ARCH_WORKAROUND_2` with `1` on kernel entry and with `0` on kernel exit.
    - Userspace mitigation via `prctl()`:
        - `PSTATE.SSBS = 0` on the current process.

### Sysfs File

#### Summary

- Access to `/sys/devices/system/cpu/vulnerabilities/spec_store_bypass` is handled by `cpu_show_spec_store_bypass()`.
- It returns one of the following three values depending on `spectre_v4_state` value:
    - `spectre_v4_state == SPECTRE_UNAFFECTED` => "Not affected"
    - `spectre_v4_state == SPECTRE_MITIGATED` => "Mitigation: Speculative Store Bypass disabled via prctl"
    - `spectre_v4_state == SPECTRE_VULNERABLE` => "Vulnerable"

#### `cpu_show_spec_store_bypass()`

[https://elixir.bootlin.com/linux/v6.11.4/source/drivers/base/cpu.c#L606](https://elixir.bootlin.com/linux/v6.11.4/source/drivers/base/cpu.c#L606)
```c
static DEVICE_ATTR(spec_store_bypass, 0444, cpu_show_spec_store_bypass, NULL);
```
The sysfs file is read only and handled by `cpu_show_spec_store_bypass()`.

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L446](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L446)
```c
ssize_t cpu_show_spec_store_bypass(struct device *dev,
				   struct device_attribute *attr, char *buf)
{
	switch (spectre_v4_state) {
	case SPECTRE_UNAFFECTED:
		return sprintf(buf, "Not affected\n");
	case SPECTRE_MITIGATED:
		return sprintf(buf, "Mitigation: Speculative Store Bypass disabled via prctl\n");
	case SPECTRE_VULNERABLE:
		fallthrough;
	default:
		return sprintf(buf, "Vulnerable\n");
	}
}
```
It is implemented in `arch/arm64/kernel/proton-pack.c` and depends on a value of `spectre_v4_state`.

The possible strings are:
- "Not affected"
- "Mitigation: Speculative Store Bypass disabled via prctl"
- "Vulnerable"

### Vulnerability / Mitigation Detection

#### Summary

- `ARM64_SPECTRE_V4` indicates that the processor is affected by Spectre-V4 (`mitigation_state` is either `SPECTRE_MITIGATED` or `SPECTRE_VULNERABLE`).
- It is detected by `has_spectre_v4()` that calls the following two functions:
    - `spectre_v4_get_cpu_hw_mitigation_state()`
    - `spectre_v4_get_cpu_fw_mitigation_state()`
- `spectre_v4_get_cpu_hw_mitigation_state()` does the following things:
    - Checks if the processor is included in the safe list => `SPECTRE_UNAFFECTED`
    - Checks if the processor supports SSBS feature (`ID_AA64PFR1_EL1.SSBS >= 1`) => `SPECTRE_MITIGATED`
    - Else => `SPECTRE_VULNERABLE`
- `spectre_v4_get_cpu_fw_mitigation_state()` does the following thigs:
    - Checks if `SMCCC_ARCH_WORKAROUND_2` is supported by calling `SMCCC_ARCH_FEATURES`. The mapping between the return value of `SMCCC_ARCH_FEATURES` and `enum mitigation_state` is as follows:
        - `SMCCC_RET_SUCCESS` => `SPECTRE_MITIGATED`
        - `SMCCC_RET_NOT_REQUIRED` => `SPECTRE_UNAFFECTED`
        - `SMCCC_ARCH_WORKAROUND_RET_UNAFFECTED` => `SPECTRE_UNAFFECTED`
        - `SMCCC_RET_NOT_SUPPORTED` => `SPECTRE_VULNERABLE`

#### `ARM64_SPECTRE_V4` errata

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/cpu_errata.c#L592](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/cpu_errata.c#L592)
```c
const struct arm64_cpu_capabilities arm64_errata[] = {
// snipped
	{
		.desc = "Spectre-v4",
		.capability = ARM64_SPECTRE_V4,
		.type = ARM64_CPUCAP_LOCAL_CPU_ERRATUM,
		.matches = has_spectre_v4,
		.cpu_enable = spectre_v4_enable_mitigation,
	},
// snipped
};
```
"Spectre-v4" is listed in `arm64_errata[]`.

Whether it is affected by Spectre-V4 is checked by `has_spectre_v4()`.

#### `has_spectre_v4()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L511](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L511)
```c
bool has_spectre_v4(const struct arm64_cpu_capabilities *cap, int scope)
{
	enum mitigation_state state;

	WARN_ON(scope != SCOPE_LOCAL_CPU || preemptible());

	state = spectre_v4_get_cpu_hw_mitigation_state();
	if (state == SPECTRE_VULNERABLE)
		state = spectre_v4_get_cpu_fw_mitigation_state();

	return state != SPECTRE_UNAFFECTED;
}
```

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/spectre.h#L23](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/spectre.h#L23)
```c
/* Watch out, ordering is important here. */
enum mitigation_state {
	SPECTRE_UNAFFECTED,
	SPECTRE_MITIGATED,
	SPECTRE_VULNERABLE,
};
```

If `state` is either `SPECTRE_MITIGATED` or `SPECTRE_VULNERABLE`, `has_spectre_v4()` returns true.
So `ARM64_SPECTRE_V4` capability indicates that the processor is affected by Spectre-V4.

#### `spectre_v4_get_cpu_hw_mitigation_state()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L466](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L466)
```c
static enum mitigation_state spectre_v4_get_cpu_hw_mitigation_state(void)
{
	// zulinx86: A safe list is hardcoded here.
	static const struct midr_range spectre_v4_safe_list[] = {
		MIDR_ALL_VERSIONS(MIDR_CORTEX_A35),
		MIDR_ALL_VERSIONS(MIDR_CORTEX_A53),
		MIDR_ALL_VERSIONS(MIDR_CORTEX_A55),
		MIDR_ALL_VERSIONS(MIDR_BRAHMA_B53),
		MIDR_ALL_VERSIONS(MIDR_QCOM_KRYO_3XX_SILVER),
		MIDR_ALL_VERSIONS(MIDR_QCOM_KRYO_4XX_SILVER),
		{ /* sentinel */ },
	};

	if (is_midr_in_range_list(read_cpuid_id(), spectre_v4_safe_list))
		return SPECTRE_UNAFFECTED;

	// zulinx86: Check ARM64_SSBS capability here.
	/* CPU features are detected first */
	if (this_cpu_has_cap(ARM64_SSBS))
		return SPECTRE_MITIGATED;

	return SPECTRE_VULNERABLE;
}
```

#### `ARM64_SSBS` feature

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/cpufeature.c#L2593](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/cpufeature.c#L2593)
```c
static const struct arm64_cpu_capabilities arm64_features[] = {
// snipped
	{
		.desc = "Speculative Store Bypassing Safe (SSBS)",
		.capability = ARM64_SSBS,
		.type = ARM64_CPUCAP_SYSTEM_FEATURE,
		.matches = has_cpuid_feature,
		ARM64_CPUID_FIELDS(ID_AA64PFR1_EL1, SSBS, IMP)
	},
// snipped
};
```
The capability `ARM64_SSBS` is listed as "Speculative Store Bypassing Safe (SSBS)" in `arm64_features[]`.

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/cpufeature.c#L165](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/cpufeature.c#L165)
```c
/*
 * ARM64_CPUID_FIELDS() encodes a field with a range from min_value to
 * an implicit maximum that depends on the sign-ess of the field.
 *
 * An unsigned field will be capped at all ones, while a signed field
 * will be limited to the positive half only.
 */
#define ARM64_CPUID_FIELDS(reg, field, min_value)			\
	__ARM64_CPUID_FIELDS(reg, field,				\
			     SYS_FIELD_VALUE(reg, field, min_value),	\
			     __ARM64_MAX_POSITIVE(reg, field))
```

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/tools/sysreg#L938](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/tools/sysreg#L938)
```c
# Sysreg 	<name>	<op0> 	<op1>	<crn>	<crm>	<op2>
# Fields	<fieldsname>
# EndSysreg
// snipped
Sysreg	ID_AA64PFR1_EL1	3	0	0	4	1
// snipped
UnsignedEnum	7:4	SSBS
	0b0000	NI
	0b0001	IMP
	0b0010	SSBS2
EndEnum
```

It checks `ID_AA64PFR1_EL1.SSBS >= 1`.

#### ID_AA64PFR1_EL1.SSBS

[Arm A-profile Architecture Registers](https://developer.arm.com/documentation/ddi0601/2024-09/AArch64-Registers/ID-AA64PFR1-EL1--AArch64-Processor-Feature-Register-1)

ID_AA64PFR1_EL1.SSBS[bits 7:4]
- 0b0000: AArch64 provides no mechanism to control the use of Speculative Store Bypassing.
- 0b0001: AArch64 provides the PSTATE.SSBS mechanism to mark regions that are Speclative Store Bypass Safe.
- 0b0010: As 0b0001, and adds the MSR and MRS instructions to directly read and write the PSTATE.SSBS field.

- FEAT_SSBS implements the functionality identified by the value 0b0001.
- FEAT_SSBS2 implements the functionality idenntified by the value 0b0010.

#### `spectre_v4_get_cpu_fw_mitigation_state()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L488](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L488)
```c
static enum mitigation_state spectre_v4_get_cpu_fw_mitigation_state(void)
{
	int ret;
	struct arm_smccc_res res;

	arm_smccc_1_1_invoke(ARM_SMCCC_ARCH_FEATURES_FUNC_ID,
			     ARM_SMCCC_ARCH_WORKAROUND_2, &res);

	ret = res.a0;
	switch (ret) {
	case SMCCC_RET_SUCCESS:
		return SPECTRE_MITIGATED;
	case SMCCC_ARCH_WORKAROUND_RET_UNAFFECTED:
		fallthrough;
	case SMCCC_RET_NOT_REQUIRED:
		return SPECTRE_UNAFFECTED;
	default:
		fallthrough;
	case SMCCC_RET_NOT_SUPPORTED:
		return SPECTRE_VULNERABLE;
	}
}
```

SMCCC stands for "Secure Monitor Call Calling Convention".

See [SMC Calling Convention (SMCCC)](https://developer.arm.com/documentation/den0028/latest) for more details.

The "1_1" part of `arm_smccc_1_1_invoke()` is the version number of SMCCC.

SMCCC_ARCH_FEATURES
- Description: Determine the availability and capability of _Arm Architecture Service_ functions
- Parameters:
    - `uint32 func_id`: Function ID of SMCCC_ARCH_FEATURES itself (0x8000_0001)
    - `uint32 arch_func_id`: Function ID of an _Arm Architecture Service_ function
- Return:
    - `int32`
        - `< 0`: Function not implemented.
            - `-1`: Not supported.
            - `-2`: Not required.
        - `0` (SUCCESS): Function implemented
        - `> 0`: Function implemented. Function capabilities are indicated using feature flags specific to the function.

SMCCC_ARCH_WORKAROUND_2
- Description: Enable or disable the mitigation for CVE-2018-3639 on the calling PE.
- Parameters:
    - `uint32 func_id`: Function ID of SMCCC_ARCH_WORKAROUND_2 (0x8000_7FFF)
    - `uint32 enable`: A non-zero value indicates that the mitigation for CVE-2018-3639 must be enabled. A value of zero indicates that it must be disabled.
- Return: `void`
- Discovery
    - The return value by SMCCC_ARCH_FEATURES is one of the following values:
        - `-1`: Not supported. SMCCC_ARCH_WORKAROUND_2 must not be invoked on any PE in the system.
            - The system contains at least 1 PE affected by CVE-2018-3639 that has no firmware mitigation available
            - The firmware does not provide any information about whether firmware mitigation is required or enabled.
        - `-2`: Not required. SMCCC_ARCH_WORKAROUND_2 must not be invoked on any PE in the system.
            - For all PEs in the system, firmware mitigation for CVE-2018-3639 is either permanently enabled or not required.
        - `0`: SMCCC_ARCH_WORKAROUND_2 can be invoked safely on all PEs in the system.
            - The PE on which SMCCC_ARCH_FEATURES is called requires dynamic firmware mitigation for CVE-2018-3639 using SMCCC_ARCH_WORKAROUND_2.
        - `1`: SMCCC_ARCH_WORKAROUND_2 can be invoked safely on all PEs in the system.
            - The PE on which SMCCC_ARCH_FEATURES is called does NOT require dynamic firmware mitigation for CVE-2018-3639 using SMCCC_ARCH_WORKAROUND_2.
            - Firmware mitigation on this PE is either permanently enabled or not required.

The return values of `arm_smccc_1_1_invoke()` used in `spectre_v4_get_cpu_fw_mitigation_state()` are:
- `SMCCC_RET_SUCCESS` (0) => `SPECTRE_MITIGATED`
- `SMCCC_ARCH_WORKAROUND_RET_UNAFFECTED` (1) => `SPECTRE_UNAFFECTED`
- `SMCCC_RET_NOT_REQUIRED` (-2) => `SPECTRE_UNAFFECTED`
- `SMCCC_RET_NOT_SUPPORTED` (-1) => `SPECTRE_VULNERABLE`

[https://elixir.bootlin.com/linux/v6.11.4/source/include/linux/arm-smccc.h#L193](https://elixir.bootlin.com/linux/v6.11.4/source/include/linux/arm-smccc.h#L193)
```c
/*
 * Return codes defined in ARM DEN 0070A
 * ARM DEN 0070A is now merged/consolidated into ARM DEN 0028 C
 */
#define SMCCC_RET_SUCCESS			0
#define SMCCC_RET_NOT_SUPPORTED			-1
#define SMCCC_RET_NOT_REQUIRED			-2
```

[https://elixir.bootlin.com/linux/v6.11.4/source/include/linux/arm-smccc.h#L127](https://elixir.bootlin.com/linux/v6.11.4/source/include/linux/arm-smccc.h#L127)
```c
#define SMCCC_ARCH_WORKAROUND_RET_UNAFFECTED	1
```

### Hardware Capability

#### KERNEL_HWCAP_SSBS

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/include/asm/hwcap.h#L91](https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/include/asm/hwcap.h#L91)
```c
/*
 * For userspace we represent hwcaps as a collection of HWCAP{,2}_x bitfields
 * as described in uapi/asm/hwcap.h. For the kernel we represent hwcaps as
 * natural numbers (in a single range of size MAX_CPU_FEATURES) defined here
 * with prefix KERNEL_HWCAP_ mapped to their HWCAP{,2}_x counterpart.
 *
 * Hwcaps should be set and tested within the kernel via the
 * cpu_{set,have}_named_feature(feature) where feature is the unique suffix
 * of KERNEL_HWCAP_{feature}.
 */
#define __khwcap_feature(x)		const_ilog2(HWCAP_ ## x)
// snipped
#define KERNEL_HWCAP_SSBS		__khwcap_feature(SSBS)
```

#### HWCAP_SSBS

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/include/uapi/asm/hwcap.h#L54](https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/include/uapi/asm/hwcap.h#L54)
```c
#define HWCAP_SSBS		(1 << 28)
```

[https://elixir.bootlin.com/linux/v6.11.5/source/Documentation/arch/arm64/elf_hwcaps.rst#L157-L158](https://elixir.bootlin.com/linux/v6.11.5/source/Documentation/arch/arm64/elf_hwcaps.rst#L157-L158)
```
HWCAP_SSBS
    Functionality implied by ID_AA64PFR1_EL1.SSBS == 0b0010.
```

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/kernel/cpufeature.c#L2989](https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/kernel/cpufeature.c#L2989)
```c
static const struct arm64_cpu_capabilities arm64_elf_hwcaps[] = {
// snipped
	HWCAP_CAP(ID_AA64PFR1_EL1, SSBS, SSBS2, CAP_HWCAP, KERNEL_HWCAP_SSBS),
// snipped
};
```

#### `hwcap_str[]`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/kernel/cpuinfo.c#L79](https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/kernel/cpuinfo.c#L79)
```c
static const char *const hwcap_str[] = {
// snipped
	[KERNEL_HWCAP_SSBS]		= "ssbs",
// snipped
};
```


### Mitigation Selection

#### Summary

- The mitigation is enabled through `spectre_v4_enable_mitigation()` that calls the following two functions:
    - `spectre_v4_enable_hw_mitigation()`
    - `spectre_v4_enable_fw_mitigation()`
- `spectre_v4_enable_hw_mitigation()`
    - Disables speculation (i.e. speculative store bypass) by setting PSTATE.SSBS and SCTLR_ELx.SSBS to 0.
    - SSBS (Speculative Store Bypass Safe) indicates that it is safe to speculate and the mitigation is not required.
- `spectre_v4_enable_fw_mitigation()`
    - Apply firmware-based mitigation by calling `SMCCC_ARCH_WORKAROUND_2`.
- The mitigation policy can be specified via `ssbd=`
    - `force-on`: Unconditionally enable mitigation for kernel and userspace.
    - `force-off`: Unconditionally disable mitigation for kernel and userspace.
    - `kernel` (aka dynamic mode): Always enable mitigation in the kernel and allow userspace to opt-in via prctl().

#### `ARM64_SPECTRE_V4` errata

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/cpu_errata.c#L592](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/cpu_errata.c#L592)
```c
const struct arm64_cpu_capabilities arm64_errata[] = {
// snipped
	{
		.desc = "Spectre-v4",
		.capability = ARM64_SPECTRE_V4,
		.type = ARM64_CPUCAP_LOCAL_CPU_ERRATUM,
		.matches = has_spectre_v4,
		.cpu_enable = spectre_v4_enable_mitigation,
	},
// snipped
};
```
The mitigation is enabled through `spectre_v4_enable_mitigation()`.

#### `spectre_v4_enable_mitigation()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L643](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L643)
```c
void spectre_v4_enable_mitigation(const struct arm64_cpu_capabilities *__unused)
{
	enum mitigation_state state;

	WARN_ON(preemptible());

	state = spectre_v4_enable_hw_mitigation();
	if (state == SPECTRE_VULNERABLE)
		state = spectre_v4_enable_fw_mitigation();

	update_mitigation_state(&spectre_v4_state, state);
}
```

It enables an appropriate mitigation through `spectre_v4_enable_hw_mitigation()` and `spectre_v4_enable_fw_mitigation()`, and then updates `spectre_v4_state` via `update_mitigation_state()`.

#### `spectre_v4_enable_hw_mitigation()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L541](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L541)
```c
static enum mitigation_state spectre_v4_enable_hw_mitigation(void)
{
	enum mitigation_state state;

	/*
	 * If the system is mitigated but this CPU doesn't have SSBS, then
	 * we must be on the safelist and there's nothing more to do.
	 */
	state = spectre_v4_get_cpu_hw_mitigation_state();
	if (state != SPECTRE_MITIGATED || !this_cpu_has_cap(ARM64_SSBS))
		return state;

	// zulinx86: ARM64_SSBS is available here.

	if (spectre_v4_mitigations_off()) {
		// zulinx86: Both PSTATE.SSBS and SCTLR_ELx.DSSBS are set to 1
		// to permit specluation, thus vulnerable.
		sysreg_clear_set(sctlr_el1, 0, SCTLR_ELx_DSSBS);
		set_pstate_ssbs(1);
		return SPECTRE_VULNERABLE;
	}

	// zulinx86: Both PSTATE.SSBS and SCTLR_ELx.DSSBSa are set to 0 not to
	// permit speculation, thus mitigated.
	/* SCTLR_EL1.DSSBS was initialised to 0 during boot */
	set_pstate_ssbs(0);

	/*
	 * SSBS is self-synchronizing and is intended to affect subsequent
	 * speculative instructions, but some CPUs can speculate with a stale
	 * value of SSBS.
	 *
	 * Mitigate this with an unconditional speculation barrier, as CPUs
	 * could mis-speculate branches and bypass a conditional barrier.
	 */
	if (IS_ENABLED(CONFIG_ARM64_ERRATUM_3194386))
		spec_bar();

	return SPECTRE_MITIGATED;
}
```

#### PSTATE.SSBS

[Arm Architecture Reference Manual for A-profile architecture](https://developer.arm.com/documentation/ddi0487/latest/)

> B2.9.1 Speculative Store Bypass Safe (SSBS)

> When the value of PSTATE.SSBS is 0, hardware is not permitted to use speculative register values in a potentially speculatively exploitable manner if the speculative read that loads the register is from earlier in the coherence order than the entry generated by the latest store to that location using the same virtual address as the load instruction.
>
> When the value of PSTATE.SSBS is 1, hardware is permitted to use speculative register values in a potentially speculatively exploitable manner if the speculative read that loads the register is from earlier in the coherence order than the entry generated by the latest store to that location using the same virtual address as the load instruction.

- `PSTATE.SSBS == 0`: Speculation (speculative store bypass) is not permitted.
- `PSTATE.SSBS == 1`: Speculation (speculative store bypass) is permitted.

#### SCTLR_ELx.DSSBS

[Arm A-profile Architecture Registers](https://developer.arm.com/documentation/ddi0601/2024-09/AArch64-Registers/SCTLR-EL1--System-Control-Register--EL1-)

SCTLR_ELx.DSSBS[bit 44] defines the default PSTATE.SSBS value on exception entry:
- 0b0: PSTATE.SSBS is set to 0 on an exception to ELx, meaning speculation is not permitted.
- 0b1: PSTATE.SSBS is set to 1 on an exception to ELx, meaning speculation is permitted.

#### `sysreg_clear_set()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/sysreg.h#L1175](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/sysreg.h#L1175)
```c
/*
 * Modify bits in a sysreg. Bits in the clear mask are zeroed, then bits in the
 * set mask are set. Other bits are left as-is.
 */
#define sysreg_clear_set(sysreg, clear, set) do {			\
	u64 __scs_val = read_sysreg(sysreg);				\
	u64 __scs_new = (__scs_val & ~(u64)(clear)) | (set);		\
	if (__scs_new != __scs_val)					\
		write_sysreg(__scs_new, sysreg);			\
} while (0)
```

`sysreg_clear_set(sctlr_el1, 0, SCTLR_ELx_DSSBS)` sets SCTRL_ELx.DDBS bit.

#### `set_pstate_ssbs()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/sysreg.h#L109](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/sysreg.h#L109)
```c
#define set_pstate_ssbs(x)		asm volatile(SET_PSTATE_SSBS(x))
```

It emits a binary code that conforms to the below format.

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/sysreg.h#L18](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/sysreg.h#L18)
```c
/*
 * ARMv8 ARM reserves the following encoding for system registers:
 * (Ref: ARMv8 ARM, Section: "System instruction class encoding overview",
 *  C5.2, version:ARM DDI 0487A.f)
 *	[20-19] : Op0
 *	[18-16] : Op1
 *	[15-12] : CRn
 *	[11-8]  : CRm
 *	[7-5]   : Op2
 */
```

#### `CONFIG_ARM64_ERRATUM_3194386`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/Kconfig#L1070](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/Kconfig#L1070)
```c
config ARM64_ERRATUM_3194386
	bool "Cortex-*/Neoverse-*: workaround for MSR SSBS not self-synchronizing"
	default y
	help
	  This option adds the workaround for the following errata:

	  * ARM Cortex-A76 erratum 3324349
	  * ARM Cortex-A77 erratum 3324348
	  * ARM Cortex-A78 erratum 3324344
	  * ARM Cortex-A78C erratum 3324346
	  * ARM Cortex-A78C erratum 3324347
	  * ARM Cortex-A710 erratam 3324338
	  * ARM Cortex-A715 errartum 3456084
	  * ARM Cortex-A720 erratum 3456091
	  * ARM Cortex-A725 erratum 3456106
	  * ARM Cortex-X1 erratum 3324344
	  * ARM Cortex-X1C erratum 3324346
	  * ARM Cortex-X2 erratum 3324338
	  * ARM Cortex-X3 erratum 3324335
	  * ARM Cortex-X4 erratum 3194386
	  * ARM Cortex-X925 erratum 3324334
	  * ARM Neoverse-N1 erratum 3324349
	  * ARM Neoverse N2 erratum 3324339
	  * ARM Neoverse-N3 erratum 3456111
	  * ARM Neoverse-V1 erratum 3324341
	  * ARM Neoverse V2 erratum 3324336
	  * ARM Neoverse-V3 erratum 3312417

	  On affected cores "MSR SSBS, #0" instructions may not affect
	  subsequent speculative instructions, which may permit unexepected
	  speculative store bypassing.

	  Work around this problem by placing a Speculation Barrier (SB) or
	  Instruction Synchronization Barrier (ISB) after kernel changes to
	  SSBS. The presence of the SSBS special-purpose register is hidden
	  from hwcaps and EL0 reads of ID_AA64PFR1_EL1, such that userspace
	  will use the PR_SPEC_STORE_BYPASS prctl to change SSBS.

	  If unsure, say Y.
```

`ARM64_ERRATUM_3194386` is an errata that may allow unexpected speculation even after setting PSTATE.SSBS to 0.

The problem can be worked around by issuing "Speculation Barrier (SB)" or "Instruction Synchronization Barrier (ISB)" after setting PSTATE.SSBS to 0.

#### `spec_bar()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/barrier.h#L43](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/barrier.h#L43)
```c
#define spec_bar()	asm volatile(ALTERNATIVE("dsb nsh\nisb\n",		\
						 SB_BARRIER_INSN"nop\n",	\
						 ARM64_HAS_SB))
```

#### `spectre_v4_enable_fw_mitigation()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L622](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L622)
```c
static enum mitigation_state spectre_v4_enable_fw_mitigation(void)
{
	enum mitigation_state state;

	state = spectre_v4_get_cpu_fw_mitigation_state();
	if (state != SPECTRE_MITIGATED)
		return state;

	// zulinx86: SMCCC_ARCH_WORKAROUND_2 mitigation is available here.

	if (spectre_v4_mitigations_off()) {
		// zulinx86: Disable SMCCC_ARCH_WORKAROUND_2 mitigation, thus
		// vulnerable.
		arm_smccc_1_1_invoke(ARM_SMCCC_ARCH_WORKAROUND_2, false, NULL);
		return SPECTRE_VULNERABLE;
	}

	// zulinx86: Enable SMCCC_ARCH_WORKAROUND_2 mitigation, thus mitigated.
	arm_smccc_1_1_invoke(ARM_SMCCC_ARCH_WORKAROUND_2, true, NULL);

	if (spectre_v4_mitigations_dynamic())
		__this_cpu_write(arm64_ssbd_callback_required, 1);

	return SPECTRE_MITIGATED;
}
```

#### `spectre_v4_mitigations_dynamic()` / `spectre_v4_mitigations_off()` / `spectre_v4_mitigations_on()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L422](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L422)
```c
/*
 * Because this was all written in a rush by people working in different silos,
 * we've ended up with multiple command line options to control the same thing.
 * Wrap these up in some helpers, which prefer disabling the mitigation if faced
 * with contradictory parameters. The mitigation is always either "off",
 * "dynamic" or "on".
 */
static bool spectre_v4_mitigations_off(void)
{
	bool ret = cpu_mitigations_off() ||
		   __spectre_v4_policy == SPECTRE_V4_POLICY_MITIGATION_DISABLED;

	if (ret)
		pr_info_once("spectre-v4 mitigation disabled by command-line option\n");

	return ret;
}

/* Do we need to toggle the mitigation state on entry to/exit from the kernel? */
static bool spectre_v4_mitigations_dynamic(void)
{
	return !spectre_v4_mitigations_off() &&
	       __spectre_v4_policy == SPECTRE_V4_POLICY_MITIGATION_DYNAMIC;
}

static bool spectre_v4_mitigations_on(void)
{
	return !spectre_v4_mitigations_off() &&
	       __spectre_v4_policy == SPECTRE_V4_POLICY_MITIGATION_ENABLED;
}
```

It returns true if the mitigation is not off and the policy is set to dynamic.

#### `__spectre_v4_policy`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L384](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L384)
```c
static enum spectre_v4_policy __read_mostly __spectre_v4_policy;

static const struct spectre_v4_param {
	const char		*str;
	enum spectre_v4_policy	policy;
} spectre_v4_params[] = {
	{ "force-on",	SPECTRE_V4_POLICY_MITIGATION_ENABLED, },
	{ "force-off",	SPECTRE_V4_POLICY_MITIGATION_DISABLED, },
	{ "kernel",	SPECTRE_V4_POLICY_MITIGATION_DYNAMIC, },
};
static int __init parse_spectre_v4_param(char *str)
{
	int i;

	if (!str || !str[0])
		return -EINVAL;

	for (i = 0; i < ARRAY_SIZE(spectre_v4_params); i++) {
		const struct spectre_v4_param *param = &spectre_v4_params[i];

		if (strncmp(str, param->str, strlen(param->str)))
			continue;

		__spectre_v4_policy = param->policy;
		return 0;
	}

	return -EINVAL;
}
early_param("ssbd", parse_spectre_v4_param);
```

[https://elixir.bootlin.com/linux/v6.11.4/source/Documentation/admin-guide/kernel-parameters.txt#L6430](https://elixir.bootlin.com/linux/v6.11.4/source/Documentation/admin-guide/kernel-parameters.txt#L6430)
```c
	ssbd=		[ARM64,HW,EARLY]
			Speculative Store Bypass Disable control

			On CPUs that are vulnerable to the Speculative
			Store Bypass vulnerability and offer a
			firmware based mitigation, this parameter
			indicates how the mitigation should be used:

			force-on:  Unconditionally enable mitigation for
				   for both kernel and userspace
			force-off: Unconditionally disable mitigation for
				   for both kernel and userspace
			kernel:    Always enable mitigation in the
				   kernel, and offer a prctl interface
				   to allow userspace to register its
				   interest in being mitigated too.
```

If `ssbd=kernel` is given, it enters the dynamic mode where always enable mitigation in the kernel and also offer a prctl interface to enable SSBD.

#### `arm64_ssbd_callback_required`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L375](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L375)
```c
/* This is the per-cpu state tracking whether we need to talk to firmware */
DEFINE_PER_CPU_READ_MOSTLY(u64, arm64_ssbd_callback_required);
```

#### `spectre_v4_state`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L373](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L373)
```c
/*
 * Spectre v4.
 *
 * If you thought Spectre v2 was nasty, wait until you see this mess. A CPU is
 * either:
 *
 * - Mitigated in hardware and listed in our "safe list".
 * - Mitigated in hardware via PSTATE.SSBS.
 * - Mitigated in software by firmware (sometimes referred to as SSBD).
 *
 * Wait, that doesn't sound so bad, does it? Keep reading...
 *
 * A major source of headaches is that the software mitigation is enabled both
 * on a per-task basis, but can also be forced on for the kernel, necessitating
 * both context-switch *and* entry/exit hooks. To make it even worse, some CPUs
 * allow EL0 to toggle SSBS directly, which can end up with the prctl() state
 * being stale when re-entering the kernel. The usual big.LITTLE caveats apply,
 * so you can have systems that have both firmware and SSBS mitigations. This
 * means we actually have to reject late onlining of CPUs with mitigations if
 * all of the currently onlined CPUs are safelisted, as the mitigation tends to
 * be opt-in for userspace. Yes, really, the cure is worse than the disease.
 *
 * The only good part is that if the firmware mitigation is present, then it is
 * present for all CPUs, meaning we don't have to worry about late onlining of a
 * vulnerable CPU if one of the boot CPUs is using the firmware mitigation.
 *
 * Give me a VAX-11/780 any day of the week...
 */
static enum mitigation_state spectre_v4_state;
```

### Speculation Control via prctl()

#### Summary

- Speculation (speculative store bypass) can be disabled or enabled via `prctl()`:
    - `PR_SPEC_ENABLE` (1): The speculation feature is enabled, mitigation is disabled.
    - `PR_SPEC_DISABLE` (2): The speculation feature is disabled, mitigation is enabled.
    - `PR_SPEC_FORCE_DISABLE` (3): Same as `PR_SPEC_DISABLE`, but cannot be undone. A subsequent `prctl(..., PR_SPEC_ENABLE)` will fail.
    - `PR_SPEC_DISABLE_NOEXEC` (4): Same as `PR_SPEC_DISABLE`, but the state will be cleared on `execve()`.
- It is done by setting PSTATE.SSSBS bit.

#### `prctl()`

[https://elixir.bootlin.com/linux/v6.11.4/source/kernel/sys.c#L2670](https://elixir.bootlin.com/linux/v6.11.4/source/kernel/sys.c#L2670)
```c
SYSCALL_DEFINE5(prctl, int, option, unsigned long, arg2, unsigned long, arg3,
		unsigned long, arg4, unsigned long, arg5)
{
// snipped
	error = 0;
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

#### `arch_prctl_spec_ctrl_set()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L763](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L763)
```c
int arch_prctl_spec_ctrl_set(struct task_struct *task, unsigned long which,
			     unsigned long ctrl)
{
	switch (which) {
	case PR_SPEC_STORE_BYPASS:
		return ssbd_prctl_set(task, ctrl);
	default:
		return -ENODEV;
	}
}
```

[https://elixir.bootlin.com/linux/v6.11.4/source/include/uapi/linux/prctl.h#L214](https://elixir.bootlin.com/linux/v6.11.4/source/include/uapi/linux/prctl.h#L214)
```c
/* Speculation control variants */
# define PR_SPEC_STORE_BYPASS		0
```

#### `ssbd_prctl_set()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L700](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L700)
```c
static int ssbd_prctl_set(struct task_struct *task, unsigned long ctrl)
{
	switch (ctrl) {
	case PR_SPEC_ENABLE:
		/* Enable speculation: disable mitigation */
		/*
		 * Force disabled speculation prevents it from being
		 * re-enabled.
		 */
		if (task_spec_ssb_force_disable(task))
			return -EPERM;

		/*
		 * If the mitigation is forced on, then speculation is forced
		 * off and we again prevent it from being re-enabled.
		 */
		if (spectre_v4_mitigations_on())
			return -EPERM;

		ssbd_prctl_disable_mitigation(task);
		break;
	case PR_SPEC_FORCE_DISABLE:
		/* Force disable speculation: force enable mitigation */
		/*
		 * If the mitigation is forced off, then speculation is forced
		 * on and we prevent it from being disabled.
		 */
		if (spectre_v4_mitigations_off())
			return -EPERM;

		task_set_spec_ssb_force_disable(task);
		fallthrough;
	case PR_SPEC_DISABLE:
		/* Disable speculation: enable mitigation */
		/* Same as PR_SPEC_FORCE_DISABLE */
		if (spectre_v4_mitigations_off())
			return -EPERM;

		ssbd_prctl_enable_mitigation(task);
		break;
	case PR_SPEC_DISABLE_NOEXEC:
		/* Disable speculation until execve(): enable mitigation */
		/*
		 * If the mitigation state is forced one way or the other, then
		 * we must fail now before we try to toggle it on execve().
		 */
		if (task_spec_ssb_force_disable(task) ||
		    spectre_v4_mitigations_off() ||
		    spectre_v4_mitigations_on()) {
			return -EPERM;
		}

		ssbd_prctl_enable_mitigation(task);
		task_set_spec_ssb_noexec(task);
		break;
	default:
		return -ERANGE;
	}

	spectre_v4_enable_task_mitigation(task);
	return 0;
}
```

[https://elixir.bootlin.com/linux/v6.11.4/source/include/uapi/linux/prctl.h#L217](https://elixir.bootlin.com/linux/v6.11.4/source/include/uapi/linux/prctl.h#L217)
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

- `PR_SPEC_ENABLE` (1): The speculation feature is enabled, mitigation is disabled.
- `PR_SPEC_DISABLE` (2): The speculation feature is disabled, mitigation is enabled.
- `PR_SPEC_FORCE_DISABLE` (3): Same as `PR_SPEC_DISABLE`, but cannot be undone. A subsequent `prctl(..., PR_SPEC_ENABLE)` will fail.
- `PR_SPEC_DISABLE_NOEXEC` (4): Same as `PR_SPEC_DISABLE`, but the state will be cleared on `execve()`.

#### `ssbd_prctl_enable_mitigation()` / `ssbd_prctl_disable_mitigation()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L679](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L679)
```c
/*
 * The Spectre-v4 mitigation can be controlled via a prctl() from userspace.
 * This is interesting because the "speculation disabled" behaviour can be
 * configured so that it is preserved across exec(), which means that the
 * prctl() may be necessary even when PSTATE.SSBS can be toggled directly
 * from userspace.
 */
static void ssbd_prctl_enable_mitigation(struct task_struct *task)
{
	task_clear_spec_ssb_noexec(task);
	task_set_spec_ssb_disable(task);
	set_tsk_thread_flag(task, TIF_SSBD);
}

static void ssbd_prctl_disable_mitigation(struct task_struct *task)
{
	task_clear_spec_ssb_noexec(task);
	task_clear_spec_ssb_disable(task);
	clear_tsk_thread_flag(task, TIF_SSBD);
}
```

#### `task{,_set,_clear}_spec_ssb_disable()` / `task{,_set,_clear}_spec_ssb_noexec()` / `task{,_set}_spec_ssb_force_disable()`

[https://elixir.bootlin.com/linux/v6.11.4/source/include/linux/sched.h#L1744](https://elixir.bootlin.com/linux/v6.11.4/source/include/linux/sched.h#L1744)
```c
/* Per-process atomic flags. */
// snipped
#define PFA_SPEC_SSB_DISABLE		3	/* Speculative Store Bypass disabled */
#define PFA_SPEC_SSB_FORCE_DISABLE	4	/* Speculative Store Bypass force disabled*/
// snipped
#define PFA_SPEC_SSB_NOEXEC		7	/* Speculative Store Bypass clear on execve() */

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

#### `{set,clear}_tsk_thread_flag()`

[https://elixir.bootlin.com/linux/v6.11.4/source/include/linux/sched.h#L1945](https://elixir.bootlin.com/linux/v6.11.4/source/include/linux/sched.h#L1945)
```c
/*
 * Set thread flags in other task's structures.
 * See asm/thread_info.h for TIF_xxxx flags available:
 */
static inline void set_tsk_thread_flag(struct task_struct *tsk, int flag)
{
	set_ti_thread_flag(task_thread_info(tsk), flag);
}

static inline void clear_tsk_thread_flag(struct task_struct *tsk, int flag)
{
	clear_ti_thread_flag(task_thread_info(tsk), flag);
}
```

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/thread_info.h#L79](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/thread_info.h#L79)
```c
#define TIF_SSBD		25	/* Wants SSB mitigation */
```

[https://elixir.bootlin.com/linux/v6.11.4/source/include/linux/thread_info.h#L87](https://elixir.bootlin.com/linux/v6.11.4/source/include/linux/thread_info.h#L87)
```c
static inline void set_ti_thread_flag(struct thread_info *ti, int flag)
{
	set_bit(flag, (unsigned long *)&ti->flags);
}

static inline void clear_ti_thread_flag(struct thread_info *ti, int flag)
{
	clear_bit(flag, (unsigned long *)&ti->flags);
}
```

#### `spectre_v4_enable_task_mitigation()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L666](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L666)
```c
void spectre_v4_enable_task_mitigation(struct task_struct *tsk)
{
	struct pt_regs *regs = task_pt_regs(tsk);
	bool ssbs = false, kthread = tsk->flags & PF_KTHREAD;

	if (spectre_v4_mitigations_off())
		ssbs = true;
	else if (spectre_v4_mitigations_dynamic() && !kthread)
		ssbs = !test_tsk_thread_flag(tsk, TIF_SSBD);

	__update_pstate_ssbs(regs, ssbs);
}
```

#### `__update_pstate_ssbs()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L656](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L656)
```c
static void __update_pstate_ssbs(struct pt_regs *regs, bool state)
{
	u64 bit = compat_user_mode(regs) ? PSR_AA32_SSBS_BIT : PSR_SSBS_BIT;

	if (state)
		regs->pstate |= bit;
	else
		regs->pstate &= ~bit;
}
```

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/ptrace.h#L61](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/asm/ptrace.h#L61)
```c
/* SPSR_ELx bits for exceptions taken from AArch32 */
// snipped
#define PSR_AA32_SSBS_BIT	0x00800000
```

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/uapi/asm/ptrace.h#L50](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/include/uapi/asm/ptrace.h#L50)
```c
/* AArch64 SPSR bits */
// snipped
#define PSR_SSBS_BIT	0x00001000
```

#### `start_thread()`

[https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/include/asm/processor.h#L298](https://elixir.bootlin.com/linux/v6.11.5/source/arch/arm64/include/asm/processor.h#L298)
```c
static inline void start_thread(struct pt_regs *regs, unsigned long pc,
				unsigned long sp)
{
// snipped
	spectre_v4_enable_task_mitigation(current);
// snipped
}
```

`spectre_v4_enable_task_mitigation()` is applied also when the thread is re-started.

### Mitigation on Kernel Entry

#### Summary

- If in the dynamic mode, SMCCC_ARCH_WORKAROUND_2 is called via `apply_ssbd` on kernel entry (`kernel_entry`) and kernel exit (`kernel_exit`).

#### `apply_ssbd`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/entry.S#L115](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/entry.S#L115)
```c
	/*
	 * This macro corrupts x0-x3. It is the caller's duty  to save/restore
	 * them if required.
	 */
	.macro	apply_ssbd, state, tmp1, tmp2
alternative_cb	ARM64_ALWAYS_SYSTEM, spectre_v4_patch_fw_mitigation_enable
	// zulinx86: In the dynamic mode, spectre_v4_patch_fw_mitigation_enable()
	// changes the following the branch to L__asm_ssbd_skip to NOP.
	b	.L__asm_ssbd_skip\@		// Patched to NOP
alternative_cb_end
	// zulinx86: arm64_ssbd_callback_required was set in 
	// spectre_v4_enable_fw_mitigation().
	ldr_this_cpu	\tmp2, arm64_ssbd_callback_required, \tmp1
	cbz	\tmp2,	.L__asm_ssbd_skip\@
	// zulinx86: Checks if TIF_SSBD flag is set.
	ldr	\tmp2, [tsk, #TSK_TI_FLAGS]
	tbnz	\tmp2, #TIF_SSBD, .L__asm_ssbd_skip\@
	// zulinx86: Calls the firmware call SMCCC_ARCH_WORKAROUND_2.
	mov	w0, #ARM_SMCCC_ARCH_WORKAROUND_2
	mov	w1, #\state
alternative_cb	ARM64_ALWAYS_SYSTEM, smccc_patch_fw_mitigation_conduit
	nop					// Patched to SMC/HVC #0
alternative_cb_end
.L__asm_ssbd_skip\@:
	.endm
```

#### `spectre_v4_patch_fw_mitigation_enable()`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L580](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/proton-pack.c#L580)
```c
/*
 * Patch a branch over the Spectre-v4 mitigation code with a NOP so that
 * we fallthrough and check whether firmware needs to be called on this CPU.
 */
void __init spectre_v4_patch_fw_mitigation_enable(struct alt_instr *alt,
						  __le32 *origptr,
						  __le32 *updptr, int nr_inst)
{
	BUG_ON(nr_inst != 1); /* Branch -> NOP */

	if (spectre_v4_mitigations_off())
		return;

	if (cpus_have_cap(ARM64_SSBS))
		return;

	if (spectre_v4_mitigations_dynamic())
		*updptr = cpu_to_le32(aarch64_insn_gen_nop());
}
```

#### `kernel_entry` / `kernel_exit`

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/entry.S#L260](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/entry.S#L260)
```c
	.macro	kernel_entry, el, regsize = 64
// snipped
	apply_ssbd 1, x22, x23
```

[https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/entry.S#L409](https://elixir.bootlin.com/linux/v6.11.4/source/arch/arm64/kernel/entry.S#L409)
```c
	.macro	kernel_exit, el
// snipped
	apply_ssbd 0, x0, x1
```



# History of Kernel Patches

Note: The lines marked * are not much related to SSB. 

## x86

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


## arm64

| Date       | No.     | Patch | Links  |
|------------|---------|-------|--------|
| 2018-05-29 | [00/17] | _arm64 SSBD (aka Spectre-v4) mitigation_ | [[lore]](https://lore.kernel.org/all/20180529121121.24927-1-marc.zyngier@arm.com/) |
| 2018-05-31 | [01/17] | arm/arm64: smccc: Add SMCCC-specific return codes | [[commit]](https://github.com/torvalds/linux/commit/eff0e9e1078ea7dc1d794dc50e31baef984c46d7) [[lore]](https://lore.kernel.org/all/20180529121121.24927-2-marc.zyngier@arm.com/) |
| 2018-05-31 | [02/17] | arm64: Call ARCH_WORKAROUND_2 on transitions between EL0 and EL1 | [[commit]](https://github.com/torvalds/linux/commit/8e2906245f1e3b0d027169d9f2e55ce0548cb96e) [[lore]](https://lore.kernel.org/all/20180529121121.24927-3-marc.zyngier@arm.com/) |
| 2018-05-31 | [03/17] | arm64: Add per-cpu infrastructure to call ARCH_WORKAROUND_2 | [[commit]](https://github.com/torvalds/linux/commit/5cf9ce6e5ea50f805c6188c04ed0daaec7b6887d) [[lore]](https://lore.kernel.org/all/20180529121121.24927-4-marc.zyngier@arm.com/) |
| 2018-05-31 | [04/17] | arm64: Add ARCH_WORKAROUND_2 probing | [[commit]](https://github.com/torvalds/linux/commit/a725e3dda1813ed306734823ac4c65ca04e38500) [[lore]](https://lore.kernel.org/all/20180529121121.24927-5-marc.zyngier@arm.com/) |
| 2018-05-31 | [05/17] | arm64: Add 'ssbd' command-line option | [[commit]](https://github.com/torvalds/linux/commit/a43ae4dfe56a01f5b98ba0cb2f784b6a43bafcc6) [[lore]](https://lore.kernel.org/all/20180529121121.24927-6-marc.zyngier@arm.com/) |
| 2018-05-31 | [06/17] | arm64: ssbd: Add global mitigation state accessor | [[commit]](https://github.com/torvalds/linux/commit/c32e1736ca03904c03de0e4459a673be194f56fd) [[lore]](https://lore.kernel.org/all/20180529121121.24927-7-marc.zyngier@arm.com/) |
| 2018-05-31 | [07/17] | arm64: ssbd: Skip apply_ssbd if not using dynamic mitigation | [[commit]](https://github.com/torvalds/linux/commit/986372c4367f46b34a3c0f6918d7fb95cbdf39d6) [[lore]](https://lore.kernel.org/all/20180529121121.24927-8-marc.zyngier@arm.com/) |
| 2018-05-31 | [08/17] | arm64: ssbd: Restore mitigation status on CPU resume | [[commit]](https://github.com/torvalds/linux/commit/647d0519b53f440a55df163de21c52a8205431cc) [[lore]](https://lore.kernel.org/all/20180529121121.24927-9-marc.zyngier@arm.com/) |
| 2018-05-31 | [09/17] | arm64: ssbd: Introduce thread flag to control userspace mitigation | [[commit]](https://github.com/torvalds/linux/commit/9dd9614f5476687abbff8d4b12cd08ae70d7c2ad) [[lore]](https://lore.kernel.org/all/20180529121121.24927-10-marc.zyngier@arm.com/) |
| 2018-05-31 | [10/17] | arm64: ssbd: Add prctl interface for per-thread mitigation | [[commit]](https://github.com/torvalds/linux/commit/9cdc0108baa8ef87c76ed834619886a46bd70cbe) [[lore]](https://lore.kernel.org/all/20180529121121.24927-11-marc.zyngier@arm.com/) |
| 2018-05-31 | [11/17] | arm64: KVM: Add HYP per-cpu accessors | [[commit]](https://github.com/torvalds/linux/commit/85478bab409171de501b719971fd25a3d5d639f9) [[lore]](https://lore.kernel.org/all/20180529121121.24927-12-marc.zyngier@arm.com/) |
| 2018-05-31 | [12/17] | arm64: KVM: Add ARCH_WORKAROUND_2 support for guests | [[commit]](https://github.com/torvalds/linux/commit/55e3748e8902ff641e334226bdcb432f9a5d78d3) [[lore]](https://lore.kernel.org/all/20180529121121.24927-13-marc.zyngier@arm.com/) |
| 2018-05-31 | [13/17] | arm64: KVM: Handle guest's ARCH_WORKAROUND_2 requests | [[commit]](https://github.com/torvalds/linux/commit/b4f18c063a13dfb33e3a63fe1844823e19c2265e) [[lore]](https://lore.kernel.org/all/20180529121121.24927-14-marc.zyngier@arm.com/) |
| 2018-05-31 | [14/17] | arm64: KVM: Add ARCH_WORKAROUND_2 discovery through ARCH_FEATURES_FUNC_ID | [[commit]](https://github.com/torvalds/linux/commit/5d81f7dc9bca4f4963092433e27b508cbe524a32) [[lore]](https://lore.kernel.org/all/20180529121121.24927-15-marc.zyngier@arm.com/) |
| 2018-08-30 | [0/7]   | _Add support for PSTATE.SSBS to mitigate Spectre-v4_ | [[lore]](https://lore.kernel.org/all/1535645767-9901-1-git-send-email-will.deacon@arm.com/) |
| 2018-09-14 | [1/7]   | arm64: Fix silly typo in comment | [[commit]](https://github.com/torvalds/linux/commit/ca7f686ac9fe87a9175696a8744e095ab9749c49) [[lore]](https://lore.kernel.org/all/1535645767-9901-2-git-send-email-will.deacon@arm.com/) |
| 2018-09-14 | [2/7]   | arm64: cpufeature: Detect SSBS and advertise to userspace | [[commit]](https://github.com/torvalds/linux/commit/d71be2b6c0e19180b5f80a6d42039cc074a693a2) [[lore]](https://lore.kernel.org/all/1535645767-9901-3-git-send-email-will.deacon@arm.com/) |
| 2018-09-14 | [3/7]   | arm64: ssbd: Drop #ifdefs for PR_SPEC_STORE_BYPASS | [[commit]](https://github.com/torvalds/linux/commit/2d1b2a91d56b19636b740ea70c8399d1df249f20) [[lore]](https://lore.kernel.org/all/1535645767-9901-4-git-send-email-will.deacon@arm.com/) |
| 2018-09-14 | [4/7]   | arm64: entry: Allow handling of undefined instructions from EL1 | [[commit]](https://github.com/torvalds/linux/commit/0bf0f444b2c49241b2b39aa3cf210d7c95ef6c34) [[lore]](https://lore.kernel.org/all/1535645767-9901-5-git-send-email-will.deacon@arm.com/) |
| 2018-09-14 | [5/7]   | arm64: ssbd: Add support for PSTATE.SSBS rather than trapping to EL3 | [[commit]](https://github.com/torvalds/linux/commit/8f04e8e6e29c93421a95b61cad62e3918425eac7) [[lore]](https://lore.kernel.org/all/1535645767-9901-6-git-send-email-will.deacon@arm.com/) |
| 2018-09-14 | [6/7]   | KVM: arm64: Set SCTLR_EL2.DSSBS if SSBD is forcefully disabled and !vhe | [[commit]](https://github.com/torvalds/linux/commit/7c36447ae5a090729e7b129f24705bb231a07e0b) [[lore]](https://lore.kernel.org/all/1535645767-9901-7-git-send-email-will.deacon@arm.com/) |
| 2018-09-14 | [7/7]   | arm64: cpu: Move errata and feature enable callbacks closer to callers | [[commit]](https://github.com/torvalds/linux/commit/b8925ee2e12d1cb9a11d6f28b5814f2bfa59dce1) [[lore]](https://lore.kernel.org/all/1535645767-9901-8-git-send-email-will.deacon@arm.com/) |
| 2020-09-18 | [00/19] | _Fix and rewrite arm64 spectre mitigations_ | [[lore]](https://lore.kernel.org/all/20200918164729.31994-1-will@kernel.org/) |
| 2020-09-29 | [03/19] | arm64: Run ARCH_WORKAROUND_2 enabling code on all CPUs | [[commit]](https://github.com/torvalds/linux/commit/39533e12063be7f55e3d6ae21ffe067799d542a4) [[lore]](https://lore.kernel.org/all/20200918164729.31994-4-will@kernel.org/) |
| 2020-09-29 | [04/19] | arm64: Remove Spectre-related CONFIG_* options | [[commit]](https://github.com/torvalds/linux/commit/6e5f0927846adf39aebee450f13871e3cb4ab012) [[lore]](https://lore.kernel.org/all/20200918164729.31994-5-will@kernel.org/) |
| 2020-09-29 | [12/19] | arm64: Treat SSBS as a non-strict system feature | [[lore]](https://lore.kernel.org/all/20200918164729.31994-13-will@kernel.org/) |
| 2020-09-29 | [13/19] | arm64: Rename ARM64_SSBD to ARM64_SPECTRE_V4 | [[commit]](https://github.com/torvalds/linux/commit/9b0955baa4208abd72840c13c5f56a0f133c7cb3) [[lore]](https://lore.kernel.org/all/20200918164729.31994-14-will@kernel.org/) |
| 2020-09-29 | [14/19] | arm64: Move SSBD prctl() handler alongside other spectre mitigation code | [[commit]](https://github.com/torvalds/linux/commit/9e78b659b4539ee20fd0c415cf8c231cea59e9c0) [[lore]](https://lore.kernel.org/all/20200918164729.31994-15-will@kernel.org/) |
| 2020-09-29 | [15/19] | arm64: Rewrite Spectre-v4 mitigation code | [[commit]](https://github.com/torvalds/linux/commit/c28762070ca651fe7a981b8f31d972c9b7d2c386) [[lore]](https://lore.kernel.org/all/20200918164729.31994-16-will@kernel.org/) |
| 2020-09-29 | [16/19] | KVM: arm64: Simplify handling of ARCH_WORKAROUND_2 | [[commit]](https://github.com/torvalds/linux/commit/29e8910a566aad3ee72f729e45842858d51ced8d) [[lore]](https://lore.kernel.org/all/20200918164729.31994-17-will@kernel.org/) |
| 2020-09-29 | [17/19] | KVM: arm64: Get rid of kvm_arm_have_ssbd() | [[commit]](https://github.com/torvalds/linux/commit/7311467702710cc30ac4e3a6c6670a766e7667f9) [[lore]](https://lore.kernel.org/all/20200918164729.31994-18-will@kernel.org/) |
| 2020-09-29 | [18/19] | KVM: arm64: Convert ARCH_WORKAROUND_2 to arm64_get_spectre_v4_state() | [[commit]](https://github.com/torvalds/linux/commit/d63d975a71b332df36cc802e6e77a462af6b9fef) [[lore]](https://lore.kernel.org/all/20200918164729.31994-19-will@kernel.org/) |
| 2020-09-29 | [19/19] | arm64: Get rid of arm64_ssbd_state | [[commit]](https://github.com/torvalds/linux/commit/31c84d6c9cde673b3866972552ec52e4c522b4fa) [[lore]](https://lore.kernel.org/all/20200918164729.31994-20-will@kernel.org/) |
| 2020-09-29 | -       | arm64: Pull in task_stack_page() to Spectre-v4 mitigation code | [[commit]](https://github.com/torvalds/linux/commit/5c8b0cbd9d6bac5f40943b5a7d8eac8cb86cbe7f) |
| 2020-09-29 | -       | arm64: Add support for PR_SPEC_DISABLE_NOEXEC prctl() option | [[commit]](https://github.com/torvalds/linux/commit/780c083a8f840ca9162c7a4090ff5e10d15152a2) |
| 2021-12-08 | -       | KVM: arm64: Drop unused workaround_flags vcpu field | [[commit]](https://github.com/torvalds/linux/commit/142ff9bddbde757674c7081ffc238cfcffa1859b) [[lore]](https://lore.kernel.org/all/164000884613.23020.16840133765876356033.tip-bot2@tip-bot2/) |



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
- ARM
    - [Speculative Processor Vulnerability](https://developer.arm.com/Arm%20Security%20Center/Speculative%20Processor%20Vulnerability)
    - [Arm Architecture Reference Manual for A-profile architecture](https://developer.arm.com/documentation/ddi0487/latest/)
    - [ID_AA64PFR1_EL1 - Arm A-profile Architecture Registers](https://developer.arm.com/documentation/ddi0601/2024-09/AArch64-Registers/ID-AA64PFR1-EL1--AArch64-Processor-Feature-Register-1)
    - [SSBS - Arm A-profile Architecture Registers](https://developer.arm.com/documentation/ddi0601/2024-09/AArch64-Registers/SSBS--Speculative-Store-Bypass-Safe?lang=en)
    - [SCTLR_EL1 - Arm A-profile Architecture Registers](https://developer.arm.com/documentation/ddi0601/2024-09/AArch64-Registers/SCTLR-EL1--System-Control-Register--EL1-)
    - [Cache Speculation Side channels v2.5](https://developer.arm.com/documentation/102816/latest/)
    - [SMC Calling Convention (SMCCC)](https://developer.arm.com/documentation/den0028/latest)
- Microsoft
    - [ADV180012 - Security Update Guide - Microsoft - Microsoft Guidance for Speculative Store Bypass](https://msrc.microsoft.com/update-guide/en-US/advisory/ADV180012)
    - [Analysis and mitigation of speculative store bypass (CVE-2018-3639) | MSRC Blog | Microsoft Security Response Center](https://msrc.microsoft.com/blog/2018/05/analysis-and-mitigation-of-speculative-store-bypass-cve-2018-3639/)
