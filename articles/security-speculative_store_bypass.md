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
        - 

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

### Vulnerability/Mitigation Detection

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


## arm64

| Date       | No      | Patch | Links  |
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
