---
title: 【Security】Spectre-RSB / PBRSB
emoji: "🛡️"
type: "idea"
topics: ["security"]
published: false
---



# General Information

- Name: Spectre-RSB, ret2spec, Return Mispredict, Post-barrier Return Stack Buffer Predictions (PBRSB)
- CVE ID: CVE-2018-15572 (Spectre-RSB), CVE-2022-26373 (PBRSB)
- Disclosure Date: August 19th, 2018 (Spectre-RSB), August 18th, 2022 (PBRSB)



# Kernel Code (v6.11.5)


## Bug Detection

### Summary

- `X86_BUG_SPECTRE_V2`: Set if the processor is not listed in the whitelist.
- `X86_BUG_EIBRS_PBRSB`: Set if the processor supports eIBRS AND the processor is not listed in the whitelist AND IA32_ARCH_CAPABILITIES.PBRSB_NO[bit 24] = 0.

### `X86_BUG_SPECTRE_V2` / `X86_BUG_EIBRS_PBRSB`

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L505
```c
#define X86_BUG_SPECTRE_V2		X86_BUG(16) /* "spectre_v2" CPU is affected by Spectre variant 2 attack with indirect branches */
// snipped
#define X86_BUG_EIBRS_PBRSB		X86_BUG(28) /* "eibrs_pbrsb" EIBRS is vulnerable to Post Barrier RSB Predictions */
```

### `cpu_set_bug_bits()`

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/common.c#L1320
```c
static void __init cpu_set_bug_bits(struct cpuinfo_x86 *c)
{
// snipped
	if (!cpu_matches(cpu_vuln_whitelist, NO_SPECTRE_V2))
		setup_force_cpu_bug(X86_BUG_SPECTRE_V2);
// snipped
	/*
	 * AMD's AutoIBRS is equivalent to Intel's eIBRS - use the Intel feature
	 * flag and protect from vendor-specific bugs via the whitelist.
	 *
	 * Don't use AutoIBRS when SNP is enabled because it degrades host
	 * userspace indirect branch performance.
	 */
	if ((x86_arch_cap_msr & ARCH_CAP_IBRS_ALL) ||
	    (cpu_has(c, X86_FEATURE_AUTOIBRS) &&
	     !cpu_feature_enabled(X86_FEATURE_SEV_SNP))) {
		setup_force_cpu_cap(X86_FEATURE_IBRS_ENHANCED);
		if (!cpu_matches(cpu_vuln_whitelist, NO_EIBRS_PBRSB) &&
		    !(x86_arch_cap_msr & ARCH_CAP_PBRSB_NO))
			setup_force_cpu_bug(X86_BUG_EIBRS_PBRSB);
	}
// snipped
}
```

### `cpu_vuln_whitelist[]`

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/common.c#L1139
```c
#define NO_SPECTRE_V2		BIT(8)
// snipped
#define NO_EIBRS_PBRSB		BIT(10)
// snipped

static const __initconst struct x86_cpu_id cpu_vuln_whitelist[] = {
// snipped
	VULNWL_INTEL(INTEL_ATOM_GOLDMONT_PLUS,	NO_MDS | NO_L1TF | NO_SWAPGS | NO_ITLB_MULTIHIT | NO_MMIO | NO_EIBRS_PBRSB),
// snipped
	VULNWL_INTEL(INTEL_ATOM_TREMONT,	NO_EIBRS_PBRSB),
	VULNWL_INTEL(INTEL_ATOM_TREMONT_L,	NO_EIBRS_PBRSB),
	VULNWL_INTEL(INTEL_ATOM_TREMONT_D,	NO_ITLB_MULTIHIT | NO_EIBRS_PBRSB),
// snipped
	/* FAMILY_ANY must be last, otherwise 0x0f - 0x12 matches won't work */
	VULNWL_AMD(X86_FAMILY_ANY,	NO_MELTDOWN | NO_L1TF | NO_MDS | NO_SWAPGS | NO_ITLB_MULTIHIT | NO_MMIO | NO_EIBRS_PBRSB | NO_BHI),
	VULNWL_HYGON(X86_FAMILY_ANY,	NO_MELTDOWN | NO_L1TF | NO_MDS | NO_SWAPGS | NO_ITLB_MULTIHIT | NO_MMIO | NO_EIBRS_PBRSB | NO_BHI),

	/* Zhaoxin Family 7 */
	VULNWL(CENTAUR,	7, X86_MODEL_ANY,	NO_SPECTRE_V2 | NO_SWAPGS | NO_MMIO | NO_BHI),
	VULNWL(ZHAOXIN,	7, X86_MODEL_ANY,	NO_SPECTRE_V2 | NO_SWAPGS | NO_MMIO | NO_BHI),
	{}
};
```


## Feature Bits

### Summary

There are three flags for RSB filling
- `X86_FEATURE_RSB_CTXSW` for context switches
- `X86_FEATURE_RSB_VMEXIT` for VM exits
- `X86_FEATURE_RSB_VMEXIT_LITE` for VM exits on parts affected by PBRSB

### `X86_FEATURE_RSB_CTXSW` / `X86_FEATURE_RSB_VMEXIT` / `X86_FEATURE_RSB_VMEXIT_LITE`

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L205
```c
/*
 * Auxiliary flags: Linux defined - For features scattered in various
 * CPUID levels like 0x6, 0xA etc, word 7.
 *
 * Reuse free bits when adding new feature flags!
 */
// snipped
#define X86_FEATURE_RSB_VMEXIT		( 7*32+13) /* Fill RSB on VM-Exit */
// snipped
#define X86_FEATURE_RSB_CTXSW		( 7*32+19) /* Fill RSB on context switches */
```

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/cpufeatures.h#L303
```c
/*
 * Extended auxiliary flags: Linux defined - for features scattered in various
 * CPUID levels like 0xf, etc.
 *
 * Reuse free bits when adding new feature flags!
 */
// snipped
#define X86_FEATURE_RSB_VMEXIT_LITE	(11*32+17) /* Fill RSB on VM exit when EIBRS is enabled */
```


## Mitigation Selection

### Summary

- Unconditional RSB filling on context switches
    - Two types of RSB-based attacks:
        - RSB underflow (aka Retbleed, Return Stack Buffer Underflow (RSBU), Branch Type Confusion (BTC)): Out of scope of this article.
        - Spectre-RSB
            - user->kernel: mitigated by SMEP or eIBRS.
                - SMEP (Supervisor Mode Execution Protection): Prevents the kernel from executing code which is user accessible.
                - eIBRS (Enhanced IBRS): eIBRS has a side effect of flushing RSB unless the processor is affected by PBRSB.
            - user->user: mitigated by RSB filling.
- Conditional RSB filling on VM exits
    - Two types of RSB-based attakcs:
        - RSB underflow: Out of scope of this article.
        - Spectre-RSB
            - If retpoline or IBRS is used for other vulnerabiliies, RSB filling is required.
            - If eIBRS is used and the processor is not affected by PBRSB, eIBRS protects against Spectre-RSB.
            - If eIBRS is used and the processor is affected by PBRSB, one-entry RSB filling is required.

### `spectre_v2_select_mitigation()`

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L1817-L1858
```c
static void __init spectre_v2_select_mitigation(void)
{
// snipped
	/*
	 * If Spectre v2 protection has been enabled, fill the RSB during a
	 * context switch.  In general there are two types of RSB attacks
	 * across context switches, for which the CALLs/RETs may be unbalanced.
	 *
	 * 1) RSB underflow
	 *
	 *    Some Intel parts have "bottomless RSB".  When the RSB is empty,
	 *    speculated return targets may come from the branch predictor,
	 *    which could have a user-poisoned BTB or BHB entry.
	 *
	 *    AMD has it even worse: *all* returns are speculated from the BTB,
	 *    regardless of the state of the RSB.
	 *
	 *    When IBRS or eIBRS is enabled, the "user -> kernel" attack
	 *    scenario is mitigated by the IBRS branch prediction isolation
	 *    properties, so the RSB buffer filling wouldn't be necessary to
	 *    protect against this type of attack.
	 *
	 *    The "user -> user" attack scenario is mitigated by RSB filling.
	 *
	 * 2) Poisoned RSB entry
	 *
	 *    If the 'next' in-kernel return stack is shorter than 'prev',
	 *    'next' could be tricked into speculating with a user-poisoned RSB
	 *    entry.
	 *
	 *    The "user -> kernel" attack scenario is mitigated by SMEP and
	 *    eIBRS.
	 *
	 *    The "user -> user" scenario, also known as SpectreBHB, requires
	 *    RSB clearing.
	 *
	 * So to mitigate all cases, unconditionally fill RSB on context
	 * switches.
	 *
	 * FIXME: Is this pointless for retbleed-affected AMD?
	 */
	setup_force_cpu_cap(X86_FEATURE_RSB_CTXSW);
	pr_info("Spectre v2 / SpectreRSB mitigation: Filling RSB on context switch\n");

	spectre_v2_determine_rsb_fill_type_at_vmexit(mode);
// snipped
}
```

### `spectre_v2_determine_rsb_fill_type_at_vmexit()`

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kernel/cpu/bugs.c#L1579-L1624
```c
static void __init spectre_v2_determine_rsb_fill_type_at_vmexit(enum spectre_v2_mitigation mode)
{
	/*
	 * Similar to context switches, there are two types of RSB attacks
	 * after VM exit:
	 *
	 * 1) RSB underflow
	 *
	 * 2) Poisoned RSB entry
	 *
	 * When retpoline is enabled, both are mitigated by filling/clearing
	 * the RSB.
	 *
	 * When IBRS is enabled, while #1 would be mitigated by the IBRS branch
	 * prediction isolation protections, RSB still needs to be cleared
	 * because of #2.  Note that SMEP provides no protection here, unlike
	 * user-space-poisoned RSB entries.
	 *
	 * eIBRS should protect against RSB poisoning, but if the EIBRS_PBRSB
	 * bug is present then a LITE version of RSB protection is required,
	 * just a single call needs to retire before a RET is executed.
	 */
	switch (mode) {
	case SPECTRE_V2_NONE:
		return;

	case SPECTRE_V2_EIBRS_LFENCE:
	case SPECTRE_V2_EIBRS:
		if (boot_cpu_has_bug(X86_BUG_EIBRS_PBRSB)) {
			setup_force_cpu_cap(X86_FEATURE_RSB_VMEXIT_LITE);
			pr_info("Spectre v2 / PBRSB-eIBRS: Retire a single CALL on VMEXIT\n");
		}
		return;

	case SPECTRE_V2_EIBRS_RETPOLINE:
	case SPECTRE_V2_RETPOLINE:
	case SPECTRE_V2_LFENCE:
	case SPECTRE_V2_IBRS:
		setup_force_cpu_cap(X86_FEATURE_RSB_VMEXIT);
		pr_info("Spectre v2 / SpectreRSB : Filling RSB on VMEXIT\n");
		return;
	}

	pr_warn_once("Unknown Spectre v2 mode, disabling RSB mitigation at VM exit");
	dump_stack();
}
```



## RSB filling

### `__switch_to_asm`: RSB filling on Context Switches

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/entry/entry_64.S#L205
```c
/*
 * %rdi: prev task
 * %rsi: next task
 */
.pushsection .text, "ax"
SYM_FUNC_START(__switch_to_asm)
// snipped
	/*
	 * When switching from a shallower to a deeper call stack
	 * the RSB may either underflow or use entries populated
	 * with userspace addresses. On CPUs where those concerns
	 * exist, overwrite the RSB with entries which capture
	 * speculative execution to prevent attack.
	 */
	FILL_RETURN_BUFFER %r12, RSB_CLEAR_LOOPS, X86_FEATURE_RSB_CTXSW
// snipped
	jmp	__switch_to
SYM_FUNC_END(__switch_to_asm)
.popsection
```

### RSB filling on VM exits

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kvm/vmx/vmenter.S#L270
```c
	/*
	 * IMPORTANT: RSB filling and SPEC_CTRL handling must be done before
	 * the first unbalanced RET after vmexit!
	 *
	 * For retpoline or IBRS, RSB filling is needed to prevent poisoned RSB
	 * entries and (in some cases) RSB underflow.
	 *
	 * eIBRS has its own protection against poisoned RSB, so it doesn't
	 * need the RSB filling sequence.  But it does need to be enabled, and a
	 * single call to retire, before the first unbalanced RET.
	 */

	FILL_RETURN_BUFFER %_ASM_CX, RSB_CLEAR_LOOPS, X86_FEATURE_RSB_VMEXIT,\
			   X86_FEATURE_RSB_VMEXIT_LITE

	pop %_ASM_ARG2	/* @flags */
	pop %_ASM_ARG1	/* @vmx */

	call vmx_spec_ctrl_restore_host
```

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/nospec-branch.h#L123
```c
#define RSB_CLEAR_LOOPS		32	/* To forcibly overwrite all entries */
```

### `FILL_RETURN_BUFFER`

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/nospec-branch.h#L257
```c
 /*
  * A simpler FILL_RETURN_BUFFER macro. Don't make people use the CPP
  * monstrosity above, manually.
  */
.macro FILL_RETURN_BUFFER reg:req nr:req ftr:req ftr2=ALT_NOT(X86_FEATURE_ALWAYS)
	ALTERNATIVE_2 "jmp .Lskip_rsb_\@", \
		__stringify(__FILL_RETURN_BUFFER(\reg,\nr)), \ftr, \
		__stringify(nop;nop;__FILL_ONE_RETURN), \ftr2

.Lskip_rsb_\@:
.endm
```

### `__FILL_RETURN_SLOT`

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/nospec-branch.h#L128
```c
/*
 * Common helper for __FILL_RETURN_BUFFER and __FILL_ONE_RETURN.
 */
#define __FILL_RETURN_SLOT			\
	ANNOTATE_INTRA_FUNCTION_CALL;		\
	call	772f;				\
	int3;					\
772:
```

### `__FILL_RETURN_BUFFER`

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/nospec-branch.h#L142
```c
/*
 * Stuff the entire RSB.
 *
 * Google experimented with loop-unrolling and this turned out to be
 * the optimal version - two calls, each with their own speculation
 * trap should their return address end up getting used, in a loop.
 */
#ifdef CONFIG_X86_64
#define __FILL_RETURN_BUFFER(reg, nr)			\
	mov	$(nr/2), reg;				\
771:							\
	__FILL_RETURN_SLOT				\
	__FILL_RETURN_SLOT				\
	add	$(BITS_PER_LONG/8) * 2, %_ASM_SP;	\
	dec	reg;					\
	jnz	771b;					\
	/* barrier for jnz misprediction */		\
	lfence;						\
	CREDIT_CALL_DEPTH				\
	CALL_THUNKS_DEBUG_INC_CTXSW
#else
/*
 * i386 doesn't unconditionally have LFENCE, as such it can't
 * do a loop.
 */
#define __FILL_RETURN_BUFFER(reg, nr)			\
	.rept nr;					\
	__FILL_RETURN_SLOT;				\
	.endr;						\
	add	$(BITS_PER_LONG/8) * nr, %_ASM_SP;
#endif
```

### `__FILL_ONE_RETURN`

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/include/asm/nospec-branch.h#L175
```c
/*
 * Stuff a single RSB slot.
 *
 * To mitigate Post-Barrier RSB speculation, one CALL instruction must be
 * forced to retire before letting a RET instruction execute.
 *
 * On PBRSB-vulnerable CPUs, it is not safe for a RET to be executed
 * before this point.
 */
#define __FILL_ONE_RETURN				\
	__FILL_RETURN_SLOT				\
	add	$(BITS_PER_LONG/8), %_ASM_SP;		\
	lfence;

#ifdef __ASSEMBLY__
```

### `vmx_spec_ctrl_restore_host()`

IBRS bit is restored by this function.

https://elixir.bootlin.com/linux/v6.11.5/source/arch/x86/kvm/vmx/vmx.c#L7227
```c
void noinstr vmx_spec_ctrl_restore_host(struct vcpu_vmx *vmx,
					unsigned int flags)
{
	u64 hostval = this_cpu_read(x86_spec_ctrl_current);

	if (!cpu_feature_enabled(X86_FEATURE_MSR_SPEC_CTRL))
		return;

	if (flags & VMX_RUN_SAVE_SPEC_CTRL)
		vmx->spec_ctrl = __rdmsr(MSR_IA32_SPEC_CTRL);

	/*
	 * If the guest/host SPEC_CTRL values differ, restore the host value.
	 *
	 * For legacy IBRS, the IBRS bit always needs to be written after
	 * transitioning from a less privileged predictor mode, regardless of
	 * whether the guest/host values differ.
	 */
	if (cpu_feature_enabled(X86_FEATURE_KERNEL_IBRS) ||
	    vmx->spec_ctrl != hostval)
		native_wrmsrl(MSR_IA32_SPEC_CTRL, hostval);

	barrier_nospec();
}
```



# History of Kernel Patches

| Date       | No.     | Patch | Links  |
|------------|---------|-------|--------|
| 2018-01-11 | [00/12] | _Retpoline: Avoid speculative indirect calls in kernel_ | [[lore]](https://lore.kernel.org/all/1515707194-20531-1-git-send-email-dwmw@amazon.co.uk/) |
| 2018-01-12 | [12/12] | x86/retpoline: Fill return stack buffer on vmexit | [[commit]](https://github.com/torvalds/linux/commit/117cc7a908c83697b0b737d15ae1eb5943afe35b) [[lore]](https://lore.kernel.org/all/1515755487-8524-1-git-send-email-dwmw@amazon.co.uk/T/#m8c66daf7c6c2f33f0ebe235cda1727906585babb) |
| 2018-01-14 | -       | x86/retpoline: Fill RSB on context switch for affected CPUs | [[commit]](https://github.com/torvalds/linux/commit/c995efd5a740d9cbafbf58bde4973e8b50b4d761) [[lore]](https://lore.kernel.org/all/1515779365-9032-1-git-send-email-dwmw@amazon.co.uk/T/#u) |
| 2018-01-27 | [0/3]   | _Speculation CPU feature cleanups_ | [[lore]](https://lore.kernel.org/all/1517070274-12128-1-git-send-email-dwmw@amazon.co.uk/) |
| 2018-01-27 | [2/3]   | x86/retpoline: Simplify vmexit_fill_RSB() | [[commit]](https://github.com/torvalds/linux/commit/1dde7415e99933bb7293d6b2843752cbdb43ec11) [[lore]](https://lore.kernel.org/all/1517070274-12128-3-git-send-email-dwmw@amazon.co.uk/) |
| 2018-01-30 | -       | x86/speculation: Protect against userspace-userspace spectreRSB | [[commit]](https://github.com/torvalds/linux/commit/fdf82a7856b32d905c39afc85e34364491e46346) [[lore]](https://lore.kernel.org/all/nycvar.YFH.7.76.1807261308190.997@cbobk.fhfr.pm/) |
| 2018-02-19 | [0/4]   | _Speculation control improvements_ | [[lore]](https://lore.kernel.org/all/1519037457-7643-1-git-send-email-dwmw@amazon.co.uk/) |
| 2018-02-20 | [3/4]   | Revert "x86/retpoline: Simplify vmexit_fill_RSB()" | [[commit]](https://github.com/torvalds/linux/commit/d1c99108af3c5992640aa2afa7d2e88c3775c06e) [[lore]](https://lore.kernel.org/all/1519037457-7643-4-git-send-email-dwmw@amazon.co.uk/) |
|            | -       | _Merge tag 'x86_bugs_retbleed' of git://git.kernel.org/pub/scm/linux/kernel/git/tip/tip_ | [[commit]](https://github.com/torvalds/linux/commit/ce114c866860aa9eae3f50974efc68241186ba60) |
| 2022-06-27 | -       | x86/speculation: Fix RSB filling with CONFIG_RETPOLINE=n | [[commit]](https://github.com/torvalds/linux/commit/b2620facef4889fefcbf2e87284f34dcd4189bce) |
| 2022-06-27 | -       | KVM: VMX: Prevent guest RSB poisoning attacks with eIBRS | [[commit]](https://github.com/torvalds/linux/commit/fc02735b14fff8c6678b521d324ade27b1a3d4cf) |
| 2022-06-27 | -       | x86/speculation: Fill RSB on vmexit for IBRS | [[commit]](https://github.com/torvalds/linux/commit/9756bba28470722dacb79ffce554336dd1f6a6cd) |
|            | -       | _Merge tag 'x86_bugs_pbrsb' of git://git.kernel.org/pub/scm/linux/kernel/git/tip/tip_ | [[commit]](https://github.com/torvalds/linux/commit/5318b987fe9f3430adb0f5d81d07052fd996835b) |
| 2022-08-03 | -       | x86/speculation: Add RSB VM Exit protections | [[commit]](https://github.com/torvalds/linux/commit/2b1299322016731d56807aa49254a5ea3080b6b3) |
| 2022-08-03 | -       | x86/speculation: Add LFENCE to RSB fill sequence | [[commit]](https://github.com/torvalds/linux/commit/ba6e31af2be96c4d0536f2152ed6f7b6c11bca47) |
|            | -       | _Merge tag 'x86-urgent-2022-08-28' of git://git.kernel.org/pub/scm/linux/kernel/git/tip/tip_ | [[commit]](https://github.com/torvalds/linux/commit/2f23a7c914317ac0b2a7e2bbe48dc00213652f98) |
| 2022-08-19 | -       | x86/nospec: Unwreck the RSB stuffing | [[commit]](https://github.com/torvalds/linux/commit/4e3aa9238277597c6c7624f302d81a7b568b6f2d) [[lore]](https://lore.kernel.org/all/YvuNdDWoUZSBjYcm@worktop.programming.kicks-ass.net/) |
| 2022-08-19 | -       | x86/nospec: Fix i386 RSB stuffing | [[commit]](https://github.com/torvalds/linux/commit/332924973725e8cdcc783c175f68cf7e162cb9e5) [[lore]](https://lore.kernel.org/all/Yv9tj9vbQ9nNlXoY@worktop.programming.kicks-ass.net) |



# References

- General
    - [NVD - cve-2018-15572](https://nvd.nist.gov/vuln/detail/cve-2018-15572)
    - [NVD - CVE-2022-26373](https://nvd.nist.gov/vuln/detail/CVE-2022-26373)
    - [ret2spec: Speculative Execution Using Return Stack Buffers](https://arxiv.org/pdf/1807.10364)
    - [Spectre Returns! Speculation Attacks using the Return Stack Buffer | USENIX](https://www.usenix.org/conference/woot18/presentation/koruyeh)
- Intel
    - [Post-barrier Return Stack Buffer Predictions / CVE-2022-26373 /...](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/advisory-guidance/post-barrier-return-stack-buffer-predictions.html)
    - [CPUID Enumeration and Architectural MSRs](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/cpuid-enumeration-and-architectural-msrs.html)
