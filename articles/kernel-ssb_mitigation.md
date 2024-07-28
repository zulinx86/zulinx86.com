---
title: 【Kernel】Speculative Store Bypass (SSB) に対する緩和策
emoji: "🐧"
type: "tech"
topics: ["linux", "kernel"]
published: true
---

[Spectre-V1 に対する緩和策](https://zenn.dev/zulinx86/articles/kernel-spectre_v1_mitigation) に続き、SSB に対しても同様にカーネルコードを読んでみる。

※ 内容の正確性は保証できません。執筆時点での最新のカーネルバージョン (6.10) を元にしています。

SSB 自体については、理解していることを前提としています。各ベンダーの資料を見ることをおすすめします。
- [Speculative Store Bypass / CVE-2018-3639 / INTEL-SA-00115](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/advisory-guidance/speculative-store-bypass.html)
- [CPUID Enumeration and Architectural MSRs](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/cpuid-enumeration-and-architectural-msrs.html)
- [White Paper | AMD64 TECHNOLOGY SPECULATIVE STORE BYPASS DISABLE](https://kib.kiev.ua/x86docs/AMD/Spectre/124441.pdf)
- [Cache Speculation Side channels v2.5](https://developer.arm.com/documentation/102816/latest/)

SSBD (Speculative Store Bypass Disable) が基本的な緩和策であり、各ベンダーごとに地味な違いがあるので、さっとまとめておく。
- Intel
    - `CPUID.(EAX=07H,ECX=0):EDX[29]`: 1 がセットされている場合、IA32_ARCH_CAPABILITIES MSR (MSR address 10AH) がサポートされている
    - `IA32_ARCH_CAPABILITIES[4]` (SSB_NO): 1 がセットされている場合、SSB に脆弱ではない
    - `CPUID.(EAX=07H,ECX=0):EDX[31]`: 1 がセットされている場合、SSBD がサポートされてる & IA32_SPEC_CTRL MSR (MSR address: 48H) がサポートされている
    - `IA32_SPEC_CTRL[2]` (SSBD): 1 をセットすると SSBD が適用される
- AMD
    - `CPUID.80000008H:EBX[26]`: 1 がセットされている場合、SSBD が必要ない
    - `CPUID.80000008H:EBX[24]`: 1 がセットされている場合、SPEC_CTL MSR (MSR address 48H) がサポートされている
    - `SPEC_CTL[2]` (SSBD): 1 をセットすると SSBD が適用される
    - `CPUID.80000008H:EBX[25]`: 1 がセットされてている場合、VIRT_SPEC_CTL MSR (MSR address C001011FH) がサポートされている (Hypervisor がゲストに対して仮想化するために存在)
    - `VIRT_SPEC_CTRL[2]` (SSBD): 1 をセットすると SSBD が適用される
    - Family 15H (Bulldozer / Piledriver / Steamroller / Excavator)
        - MSR C0011020H の bit 54 をセットすると SSBD を適用できる
    - Family 16H (Jaguar / Puma)
        - MSR C0011020H の bit 33 をセットすると SSBD を適用できる
    - Family 17H (Zen / Zen+ / Zen 2)
        - 1 論理プロセッサのみをサポートしている場合、MSR C0011020H の bit 10 をセットすると SSBD を適用できる
        - 2 論理プロセッサをサポートしている場合、同一コア上で MSR が共有されている場合、もう一方の論理プロセッサが勝手に SSBD を無効にしないようにしないといけない。
- ARM
    - SSBB barrier: SSBB 前にある仮想アドレスを使った書き込みは、SSBB 以降にある同一仮想アドレスを使った読み込みにバイパスされない
    - PSSBB barrier: PSSBB 前にある物理アドレスを使った書き込みは、PSSBB 以降にある同一物理アドレスを使った読み込みにバイパスされない



# x86

以下のパッチセットが、Linus によって取り込まれたパッチセットの模様。

https://lore.kernel.org/all/alpine.DEB.2.21.1805211542220.1599@nanos.tec.linutronix.de/

パッチのリストはあるが、順番がわからないので、他のブランチへ backport されたやつを見てみる。

https://lore.kernel.org/all/20180521210515.145840770@linuxfoundation.org/

以下の順番で取り込まれている模様。

```
x86/nospec: Simplify alternative_msr_write()
x86/bugs: Concentrate bug detection into a separate function
x86/bugs: Concentrate bug reporting into a separate function
x86/bugs: x86/bugs: Read SPEC_CTRL MSR during boot and re-use reserved bits
x86/bugs, KVM: Support the combination of guest and host IBRS
x86/bugs: Expose /sys/../spec_store_bypass
x86/cpufeatures: Add X86_FEATURE_RDS
x86/bugs: Provide boot parameters for the spec_store_bypass_disable mitigation
x86/bugs/intel: Set proper CPU features and setup RDS
x86/bugs: Whitelist allowed SPEC_CTRL MSR values
x86/bugs/AMD: Add support to disable RDS on Fam[15,16,17]h if requested
x86/KVM/VMX: Expose SPEC_CTRL Bit(2) to the guest
x86/speculation: Create spec-ctrl.h to avoid include hell
prctl: Add speculation control prctls
x86/process: Allow runtime control of Speculative Store Bypass
x86/speculation: Add prctl for Speculative Store Bypass mitigation
nospec: Allow getting/setting on non-current task
proc: Provide details on speculation flaw mitigations
seccomp: Enable "
x86/bugs: Make boot modes __ro_after_init
prctl: Add force disable speculation
seccomp: Use PR_SPEC_FORCE_DISABLE
seccomp: Add filter flag to opt-out of SSB mitigation
seccomp: Move speculation migitation control to arch code
x86/speculation: Make "seccomp" the default mode for Speculative Store Bypass
x86/bugs: Rename _RDS to _SSBD
proc: Use underscores for SSBD in status
Documentation/spec_ctrl: Do some minor cleanups
x86/bugs: Fix __ssb_select_mitigation() return type
x86/bugs: Make cpu_show_common() static
x86/bugs: Fix the parameters alignment and missing void
x86/cpu: Make alternative_msr_write work for 32-bit code
KVM: SVM: Move spec control call after restore of GS
x86/speculation: Use synthetic bits for IBRS/IBPB/STIBP
x86/cpufeatures: Disentangle MSR_SPEC_CTRL enumeration from IBRS
x86/cpufeatures: Disentangle SSBD enumeration
x86/cpufeatures: Add FEATURE_ZEN
x86/speculation: Handle HT correctly on AMD
x86/bugs, KVM: Extend speculation control for VIRT_SPEC_CTRL
x86/speculation: Add virtualized speculative store bypass disable support
x86/speculation: Rework speculative_store_bypass_update()
x86/bugs: Unify x86_spec_ctrl_{set_guest,restore_host}
x86/bugs: Expose x86_spec_ctrl_base directly
x86/bugs: Remove x86_spec_ctrl_set()
x86/bugs: Rework spec_ctrl base and mask logic
x86/speculation, KVM: Implement support for VIRT_SPEC_CTRL/LS_CFG
KVM: SVM: Implement VIRT_SPEC_CTRL support for SSBD
x86/bugs: Rename SSBD_NO to SSB_NO
bpf: Prevent memory disambiguation attack
```

中身を見てみると、他の脆弱性に関するものも混ざっている。

ざっと抜き出すと、以下が SSB に関連するパッチな模様。

- [x86/bugs: Expose /sys/../spec_store_bypass](https://github.com/torvalds/linux/commit/c456442cd3a59eeb1d60293c26cbe2ff2c4e42cf)
- [x86/cpufeatures: Add X86_FEATURE_RDS](https://github.com/torvalds/linux/commit/0cc5fa00b0a88dad140b4e5c2cead9951ad36822)
- [x86/bugs: Provide boot parameters for the spec_store_bypass_disable mitigation](https://github.com/torvalds/linux/commit/24f7fc83b9204d20f878c57cb77d261ae825e033)
- [x86/bugs/intel: Set proper CPU features and setup RDS](https://github.com/torvalds/linux/commit/772439717dbf703b39990be58d8d4e3e4ad0598a)
- [x86/bugs: Whitelist allowed SPEC_CTRL MSR values](https://github.com/torvalds/linux/commit/1115a859f33276fe8afb31c60cf9d8e657872558)
- [x86/bugs/AMD: Add support to disable RDS on Fam[15,16,17]h if requested](https://github.com/torvalds/linux/commit/764f3c21588a059cd783c6ba0734d4db2d72822d)
- [x86/KVM/VMX: Expose SPEC_CTRL Bit(2) to the guest](https://github.com/torvalds/linux/commit/da39556f66f5cfe8f9c989206974f1cb16ca5d7c)
- [x86/speculation: Create spec-ctrl.h to avoid include hell](https://github.com/torvalds/linux/commit/28a2775217b17208811fa43a9e96bd1fdf417b86)
- [prctl: Add speculation control prctls](https://github.com/torvalds/linux/commit/b617cfc858161140d69cc0b5cc211996b557a1c7)
- [x86/process: Allow runtime control of Speculative Store Bypass](https://github.com/torvalds/linux/commit/885f82bfbc6fefb6664ea27965c3ab9ac4194b8c)
- [x86/speculation: Add prctl for Speculative Store Bypass mitigation](https://github.com/torvalds/linux/commit/a73ec77ee17ec556fe7f165d00314cb7c047b1ac)
- [nospec: Allow getting/setting on non-current task](https://github.com/torvalds/linux/commit/7bbf1373e228840bb0295a2ca26d548ef37f448e)
- [proc: Provide details on speculation flaw mitigations](https://github.com/torvalds/linux/commit/fae1fa0fc6cca8beee3ab8ed71d54f9a78fa3f64)
- [seccomp: Enable speculation flaw mitigations](https://github.com/torvalds/linux/commit/5c3070890d06ff82eecb808d02d2ca39169533ef)
- [x86/bugs: Make boot modes __ro_after_init](https://github.com/torvalds/linux/commit/f9544b2b076ca90d887c5ae5d74fab4c21bb7c13)
- [prctl: Add force disable speculation](https://github.com/torvalds/linux/commit/356e4bfff2c5489e016fdb925adbf12a1e3950ee)
- [seccomp: Use PR_SPEC_FORCE_DISABLE](https://github.com/torvalds/linux/commit/b849a812f7eb92e96d1c8239b06581b2cfd8b275)
- [seccomp: Add filter flag to opt-out of SSB mitigation](https://github.com/torvalds/linux/commit/00a02d0c502a06d15e07b857f8ff921e3e402675)
- [seccomp: Move speculation migitation control to arch code](https://github.com/torvalds/linux/commit/8bf37d8c067bb7eb8e7c381bdadf9bd89182b6bc)
- [x86/speculation: Make "seccomp" the default mode for Speculative Store Bypass](https://github.com/torvalds/linux/commit/f21b53b20c754021935ea43364dbf53778eeba32)
- [x86/bugs: Rename _RDS to _SSBD](https://github.com/torvalds/linux/commit/9f65fb29374ee37856dbad847b4e121aab72b510)
- [proc: Use underscores for SSBD in status](https://github.com/torvalds/linux/commit/e96f46ee8587607a828f783daa6eb5b44d25004d)
- [Documentation/spec_ctrl: Do some minor cleanups](https://github.com/torvalds/linux/commit/dd0792699c4058e63c0715d9a7c2d40226fcdddc)
- [x86/bugs: Fix __ssb_select_mitigation() return type](https://github.com/torvalds/linux/commit/d66d8ff3d21667b41eddbe86b35ab411e40d8c5f)
- [x86/bugs: Make cpu_show_common() static](https://github.com/torvalds/linux/commit/7bb4d366cba992904bffa4820d24e70a3de93e76)
- [x86/bugs: Fix the parameters alignment and missing void](https://github.com/torvalds/linux/commit/ffed645e3be0e32f8e9ab068d257aee8d0fe8eec)
- [x86/cpu: Make alternative_msr_write work for 32-bit code](https://github.com/torvalds/linux/commit/5f2b745f5e1304f438f9b2cd03ebc8120b6e0d3b)
- [KVM: SVM: Move spec control call after restore of GS](https://github.com/torvalds/linux/commit/15e6c22fd8e5a42c5ed6d487b7c9fe44c2517765)
- [x86/speculation: Use synthetic bits for IBRS/IBPB/STIBP](https://github.com/torvalds/linux/commit/e7c587da125291db39ddf1f49b18e5970adbac17)
- [x86/cpufeatures: Disentangle MSR_SPEC_CTRL enumeration from IBRS](https://github.com/torvalds/linux/commit/7eb8956a7fec3c1f0abc2a5517dada99ccc8a961)
- [x86/cpufeatures: Disentangle SSBD enumeration](https://github.com/torvalds/linux/commit/52817587e706686fcdb27f14c1b000c92f266c96)
- [x86/cpufeatures: Add FEATURE_ZEN](https://github.com/torvalds/linux/commit/d1035d971829dcf80e8686ccde26f94b0a069472)
- [x86/speculation: Handle HT correctly on AMD](https://github.com/torvalds/linux/commit/1f50ddb4f4189243c05926b842dc1a0332195f31)
- [x86/bugs, KVM: Extend speculation control for VIRT_SPEC_CTRL](https://github.com/torvalds/linux/commit/ccbcd2674472a978b48c91c1fbfb66c0ff959f24)
- [x86/speculation: Add virtualized speculative store bypass disable support](https://github.com/torvalds/linux/commit/11fb0683493b2da112cd64c9dada221b52463bf7)
- [x86/speculation: Rework speculative_store_bypass_update()](https://github.com/torvalds/linux/commit/0270be3e34efb05a88bc4c422572ece038ef3608)
- [x86/bugs: Unify x86_spec_ctrl_{set_guest,restore_host}](https://github.com/torvalds/linux/commit/cc69b34989210f067b2c51d5539b5f96ebcc3a01)
- [x86/bugs: Expose x86_spec_ctrl_base directly](https://github.com/torvalds/linux/commit/fa8ac4988249c38476f6ad678a4848a736373403)
- [x86/bugs: Remove x86_spec_ctrl_set()](https://github.com/torvalds/linux/commit/4b59bdb569453a60b752b274ca61f009e37f4dae)
- [x86/bugs: Rework spec_ctrl base and mask logic](https://github.com/torvalds/linux/commit/be6fcb5478e95bb1c91f489121238deb3abca46a)
- [x86/speculation, KVM: Implement support for VIRT_SPEC_CTRL/LS_CFG](https://github.com/torvalds/linux/commit/47c61b3955cf712cadfc25635bf9bc174af030ea)
- [KVM: SVM: Implement VIRT_SPEC_CTRL support for SSBD](https://github.com/torvalds/linux/commit/bc226f07dcd3c9ef0b7f6236fe356ea4a9cb4769)
- [x86/bugs: Rename SSBD_NO to SSB_NO](https://github.com/torvalds/linux/commit/240da953fcc6a9008c92fae5b1f727ee5ed167ab)
- [bpf: Prevent memory disambiguation attack](https://lore.kernel.org/all/20180521210515.145840770@linuxfoundation.org/)

最初は SSBD は RDS (Reduced Data Speculation) と呼ばれていた模様。

最初のパッチでどういうコードが追加されたかを確認すると、最新コードでどの辺を中心に読んだら良さそうかわかりやすくなる。


## Bug Detection

### [`cpu_set_bug_bits()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/common.c#L1320)

SSB に影響を受けているかどうかを確認して、影響する場合は `X86_BUG_SPEC_STORE_BYPASS` をセットする。

`X86_BUG_*` は、現在使われている CPU がどの脆弱性に脆弱かを記録するためのビットである。
脆弱かどうかが、Intel と AMD で異なる CPUID で示されるため、後の処理を共通化するために、このような合成ビットを定義している。

最初の 2 つの条件は Intel 用、最後の 1 つは AMD 用。詳細は後述。

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

### [`X86_BUG_SPEC_STORE_BYPASS`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/cpufeatures.h#L502)

```c
#define X86_BUG_SPEC_STORE_BYPASS	X86_BUG(17) /* CPU is affected by speculative store bypass attack */
```

### [`cpu_vuln_whitelist[]`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/common.c#L1139)

SSB の影響を受けないと知られている CPU のリストが定義されている。

```c
static const __initconst struct x86_cpu_id cpu_vuln_whitelist[] = {
// snipped
	VULNWL_INTEL(INTEL_ATOM_SILVERMONT,	NO_SSB | NO_L1TF | MSBDS_ONLY | NO_SWAPGS | NO_ITLB_MULTIHIT),
	VULNWL_INTEL(INTEL_ATOM_SILVERMONT_D,	NO_SSB | NO_L1TF | MSBDS_ONLY | NO_SWAPGS | NO_ITLB_MULTIHIT),
	VULNWL_INTEL(INTEL_ATOM_SILVERMONT_MID,	NO_SSB | NO_L1TF | MSBDS_ONLY | NO_SWAPGS | NO_ITLB_MULTIHIT),
	VULNWL_INTEL(INTEL_ATOM_AIRMONT,	NO_SSB | NO_L1TF | MSBDS_ONLY | NO_SWAPGS | NO_ITLB_MULTIHIT),
	VULNWL_INTEL(INTEL_XEON_PHI_KNL,	NO_SSB | NO_L1TF | MSBDS_ONLY | NO_SWAPGS | NO_ITLB_MULTIHIT),
	VULNWL_INTEL(INTEL_XEON_PHI_KNM,	NO_SSB | NO_L1TF | MSBDS_ONLY | NO_SWAPGS | NO_ITLB_MULTIHIT),
// snipped
	/* AMD Family 0xf - 0x12 */
	VULNWL_AMD(0x0f,	NO_MELTDOWN | NO_SSB | NO_L1TF | NO_MDS | NO_SWAPGS | NO_ITLB_MULTIHIT | NO_MMIO | NO_BHI),
	VULNWL_AMD(0x10,	NO_MELTDOWN | NO_SSB | NO_L1TF | NO_MDS | NO_SWAPGS | NO_ITLB_MULTIHIT | NO_MMIO | NO_BHI),
	VULNWL_AMD(0x11,	NO_MELTDOWN | NO_SSB | NO_L1TF | NO_MDS | NO_SWAPGS | NO_ITLB_MULTIHIT | NO_MMIO | NO_BHI),
	VULNWL_AMD(0x12,	NO_MELTDOWN | NO_SSB | NO_L1TF | NO_MDS | NO_SWAPGS | NO_ITLB_MULTIHIT | NO_MMIO | NO_BHI),
// snipped
};
```

### [`x86_read_arch_cap_msr()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/common.c#L1285)

IA32_ARCH_CAPABILITIES MSR がサポートされている場合に、それを読み込んで返す。

```c
u64 x86_read_arch_cap_msr(void)
{
	u64 x86_arch_cap_msr = 0;

	if (boot_cpu_has(X86_FEATURE_ARCH_CAPABILITIES))
		rdmsrl(MSR_IA32_ARCH_CAPABILITIES, x86_arch_cap_msr);

	return x86_arch_cap_msr;
}
```

[https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/cpufeatures.h#L436](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/cpufeatures.h#L436)
```c
/* Intel-defined CPU features, CPUID level 0x00000007:0 (EDX), word 18 */
// snipped
#define X86_FEATURE_ARCH_CAPABILITIES	(18*32+29) /* IA32_ARCH_CAPABILITIES MSR (Intel) */
```

### [`ARCH_CAP_SSB_NO`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/msr-index.h#L116)

```c
#define MSR_IA32_ARCH_CAPABILITIES	0x0000010a
// snipped
#define ARCH_CAP_SSB_NO			BIT(4)	/*
						 * Not susceptible to Speculative Store Bypass
						 * attack, so no Speculative Store Bypass
						 * control required.
						 */
```

### [`X86_FEATURE_AMD_SSB_NO`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/cpufeatures.h#L347)

```c
/* AMD-defined CPU features, CPUID level 0x80000008 (EBX), word 13 */
// snipped
#define X86_FEATURE_AMD_SSB_NO		(13*32+26) /* "" Speculative Store Bypass is fixed in hardware. */
```


## SSBD Feature Detection

### [`init_speculation_control()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/common.c#L931)

BUG ビットと同様に、SSBD のサポートも Intel と AMD で違う CPUID で示されるので、合成ビットが定義されている。

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

[https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/cpufeatures.h](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/cpufeatures.h)

```c
/* AMD-defined CPU features, CPUID level 0x80000008 (EBX), word 13 */
// snipped
#define X86_FEATURE_VIRT_SSBD		(13*32+25) /* Virtualized Speculative Store Bypass Disable */
// snipped
#define X86_FEATURE_AMD_SSBD		(13*32+24) /* "" Speculative Store Bypass Disable */
```
```c
/* Intel-defined CPU features, CPUID level 0x00000007:0 (EDX), word 18 */
// snipped
#define X86_FEATURE_SPEC_CTRL_SSBD	(18*32+31) /* "" Speculative Store Bypass Disable */
```
```c
/*
 * Auxiliary flags: Linux defined - For features scattered in various
 * CPUID levels like 0x6, 0xA etc, word 7.
 *
 * Reuse free bits when adding new feature flags!
 */
// snipped
#define X86_FEATURE_MSR_SPEC_CTRL	( 7*32+16) /* "" MSR SPEC_CTRL is implemented */
#define X86_FEATURE_SSBD		( 7*32+17) /* Speculative Store Bypass Disable */
```


## Mitigation Selection

### [`cpu_select_mitigations()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L166)

SSB の緩和策を選択する関数を呼び出しているところ。

```c
void __init cpu_select_mitigations(void)
{
// snipped
	ssb_select_mitigation();
// snipped
}
```

### [`ssb_select_mitigation()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L2108)

```c
static enum ssb_mitigation ssb_mode __ro_after_init = SPEC_STORE_BYPASS_NONE;

// snipped

static void ssb_select_mitigation(void)
{
	ssb_mode = __ssb_select_mitigation();

	if (boot_cpu_has_bug(X86_BUG_SPEC_STORE_BYPASS))
		pr_info("%s\n", ssb_strings[ssb_mode]);
}
```

### [`enum ssb_mitigation`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/nospec-branch.h#L505)

以下の 4 つの緩和策から 1 つが選ばれる。

```c
/* The Speculative Store Bypass disable variants */
enum ssb_mitigation {
	SPEC_STORE_BYPASS_NONE,
	SPEC_STORE_BYPASS_DISABLE,
	SPEC_STORE_BYPASS_PRCTL,
	SPEC_STORE_BYPASS_SECCOMP,
};
```

### [`ssb_strings[]`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L1998)

各緩和策に対応する sysfs の文字列。

注: `SPEC_STORE_BYPASS_SECCOMP` は seccomp だけでなく prctl も含む。

```c
static const char * const ssb_strings[] = {
	[SPEC_STORE_BYPASS_NONE]	= "Vulnerable",
	[SPEC_STORE_BYPASS_DISABLE]	= "Mitigation: Speculative Store Bypass disabled",
	[SPEC_STORE_BYPASS_PRCTL]	= "Mitigation: Speculative Store Bypass disabled via prctl",
	[SPEC_STORE_BYPASS_SECCOMP]	= "Mitigation: Speculative Store Bypass disabled via prctl and seccomp",
};
```

### [`__ssb_select_mitigation()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L2048)

kernel コマンドラインに応じて選択される。
一度コマンドラインの処理を見て、その後で戻ってきます。

```c
static enum ssb_mitigation __init __ssb_select_mitigation(void)
{
	enum ssb_mitigation mode = SPEC_STORE_BYPASS_NONE;
	enum ssb_mitigation_cmd cmd;

// snipped
	cmd = ssb_parse_cmdline();
// snipped

	switch (cmd) {
// snipped
	}

// snipped
	return mode;
}
```

### [`ssb_parse_cmdline()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L2016)

kernel コマンドラインにある文字列から `enum ssb_mitigation_cmd` へのマッピング。
- `SPEC_STORE_BYPASS_CMD_NONE`: "nospec_store_bypass_disable" or "spec_store_bypass_disable=off"
- `SPEC_STORE_BYPASS_AUTO`: "spec_store_bypass_disable=auto" or "spec_store_bypass_disable" が指定されていない
- `SPEC_STORE_BYPASS_ON`: "spec_store_bypass_disable=on"
- `SPEC_STORE_BYPASS_PRCTL`: "spec_store_bypass_disable=prctl"
- `SPEC_STORE_BYPASS_SECCOMP`: "spec_store_bypass_disable=seccomp"

```c
/* The kernel command line selection */
enum ssb_mitigation_cmd {
	SPEC_STORE_BYPASS_CMD_NONE,
	SPEC_STORE_BYPASS_CMD_AUTO,
	SPEC_STORE_BYPASS_CMD_ON,
	SPEC_STORE_BYPASS_CMD_PRCTL,
	SPEC_STORE_BYPASS_CMD_SECCOMP,
};

// snipped

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

### [`__ssb_select_mitigation()` Part.2](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L2048)

基本的には kernel コマンドラインで指定された通りになるが、SSBD がサポートされていない、SSB 自体に影響を受けない、seccomp のサポート可否に応じて変化する。

- SSBD 機能を持ってない => `SPEC_STORE_BYPASS_NONE`
- SSB に影響を受けない and (`SPEC_STORE_BYPASS_CMD_NONE` or `SPEC_STORE_BYPASS_CMD_AUTO`) => `SPEC_STORE_BYPASS_NONE`
- `SPEC_STORE_BYPASS_CMD_SECCOMP` and `CONFIG_SECCOMP` enabled => `SPEC_STORE_BYPASS_SECCOMP`
- `SPEC_STORE_BYPASS_CMD_SECCOMP` and `CONFIG_SECCOMP` disabled => `SPEC_STORE_BYPASS_PRCTL`
- `SPEC_STORE_BYPASS_CMD_ON` => `SPEC_STORE_BYPASS_DISABLE`
- `SPEC_STORE_BYPASS_CMD_AUTO` => `SPEC_STORE_BYPASS_PRCTL`
- `SPEC_STORE_BYPASS_CMD_PRCTL` => `SPEC_STORE_BYPASS_PRCTL`
- `SPEC_STORE_BYPASS_CMD_OFF` => `SPEC_STORE_BYPASS_NONE`

フルで SSBD を有効にしている場合は `X86_FEATURE_SPEC_STORE_BYPASS_DISABLE` をセットして、その場で有効化する。
IA32_SPEC_CTRL MSR がサポートされている場合とそうでは場合で処理が分かれる。

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

### [`X86_FEATURE_SPEC_STORE_BYPASS_DISABLE`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/cpufeatures.h#L215)

```c
#define X86_FEATURE_SPEC_STORE_BYPASS_DISABLE	( 7*32+23) /* "" Disable Speculative Store Bypass. */
```

### [`x86_amd_ssb_disable()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L222)

IA32_SPEC_CTRL MSR がサポートされていない AMD での処理。
古い AMD しか対象ではないので、ここではスキップする。

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

### [`x86_spec_ctrl_base`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L54)

```c
/* The base value of the SPEC_CTRL MSR without task-specific bits set */
u64 x86_spec_ctrl_base;
EXPORT_SYMBOL_GPL(x86_spec_ctrl_base);
```

緩和策の選択をする前に IA32_SPEC_CTRL MSR の値を読み込んで、`x86_spec_ctrl_base` に保存している。

```c
void __init cpu_select_mitigations(void)
{
	/*
	 * Read the SPEC_CTRL MSR to account for reserved bits which may
	 * have unknown values. AMD64_LS_CFG MSR is cached in the early AMD
	 * init code as it is not enumerated and depends on the family.
	 */
	if (cpu_feature_enabled(X86_FEATURE_MSR_SPEC_CTRL)) {
		rdmsrl(MSR_IA32_SPEC_CTRL, x86_spec_ctrl_base);

		/*
		 * Previously running kernel (kexec), may have some controls
		 * turned ON. Clear them and let the mitigations setup below
		 * rediscover them based on configuration.
		 */
		x86_spec_ctrl_base &= ~SPEC_CTRL_MITIGATIONS_MASK;
	}
// snipped
}
```

### [`SPEC_CTRL_MITIGFATIONS_MASK`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/msr-index.h#L68)

```c
/* A mask for bits which the kernel toggles when controlling mitigations */
#define SPEC_CTRL_MITIGATIONS_MASK	(SPEC_CTRL_IBRS | SPEC_CTRL_STIBP | SPEC_CTRL_SSBD \
							| SPEC_CTRL_RRSBA_DIS_S \
							| SPEC_CTRL_BHI_DIS_S)
```

### [`update_spec_ctrl()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L71)

```c
/* The current value of the SPEC_CTRL MSR with task-specific bits set */
DEFINE_PER_CPU(u64, x86_spec_ctrl_current);
EXPORT_PER_CPU_SYMBOL_GPL(x86_spec_ctrl_current);

// snipped

/* Update SPEC_CTRL MSR and its cached copy unconditionally */
static void update_spec_ctrl(u64 val)
{
	this_cpu_write(x86_spec_ctrl_current, val);
	wrmsrl(MSR_IA32_SPEC_CTRL, val);
}
```


## sysfs

### [`spec_store_bypass`](https://elixir.bootlin.com/linux/v6.10/source/drivers/base/cpu.c#L596)

```c
static DEVICE_ATTR(spec_store_bypass, 0444, cpu_show_spec_store_bypass, NULL);
```

### [`cpu_show_spec_store_bypass()`](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/kernel/cpu/bugs.c#L2962)

```c
ssize_t cpu_show_spec_store_bypass(struct device *dev, struct device_attribute *attr, char *buf)
{
	return cpu_show_common(dev, attr, buf, X86_BUG_SPEC_STORE_BYPASS);
}
```

### [`cpu_show_common()`](https://elixir.bootlin.com/linux/v6.10/source/arch/x86/kernel/cpu/bugs.c#L2882)

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

### [`ssb_strings[]`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L1998)

再掲。

```c
static const char * const ssb_strings[] = {
	[SPEC_STORE_BYPASS_NONE]	= "Vulnerable",
	[SPEC_STORE_BYPASS_DISABLE]	= "Mitigation: Speculative Store Bypass disabled",
	[SPEC_STORE_BYPASS_PRCTL]	= "Mitigation: Speculative Store Bypass disabled via prctl",
	[SPEC_STORE_BYPASS_SECCOMP]	= "Mitigation: Speculative Store Bypass disabled via prctl and seccomp",
};
```

## prctl

https://docs.kernel.org/userspace-api/spec_ctrl.html

パフォーマンスの影響が大きい緩和策では、ユーザーがプロセスやタスクレベルで細かく有効・無効にできるようになっている。

> There is also a class of mitigations which are very expensive, but they can be restricted to a certain set of processes or tasks in controlled environments. The mechanism to control these mitigations is via prctl(2).

以下のように `prctl()` を使って、制御することができる。

> PR_SPEC_STORE_BYPASS: Speculative Store Bypass
> Invocations:
> - prctl(PR_GET_SPECULATION_CTRL, PR_SPEC_STORE_BYPASS, 0, 0, 0);
> - prctl(PR_SET_SPECULATION_CTRL, PR_SPEC_STORE_BYPASS, PR_SPEC_ENABLE, 0, 0);
> - prctl(PR_SET_SPECULATION_CTRL, PR_SPEC_STORE_BYPASS, PR_SPEC_DISABLE, 0, 0);
> - prctl(PR_SET_SPECULATION_CTRL, PR_SPEC_STORE_BYPASS, PR_SPEC_FORCE_DISABLE, 0, 0);
> - prctl(PR_SET_SPECULATION_CTRL, PR_SPEC_STORE_BYPASS, PR_SPEC_DISABLE_NOEXEC, 0, 0);

### [`prctl()`](https://elixir.bootlin.com/linux/v6.10.2/source/kernel/sys.c#L2457)

```c
SYSCALL_DEFINE5(prctl, int, option, unsigned long, arg2, unsigned long, arg3,
		unsigned long, arg4, unsigned long, arg5)
{
	struct task_struct *me = current;
// snipped
	switch (option) {
// snipped
	case PR_GET_SPECULATION_CTRL:
		if (arg3 || arg4 || arg5)
			return -EINVAL;
		error = arch_prctl_spec_ctrl_get(me, arg2);
		break;
	case PR_SET_SPECULATION_CTRL:
		if (arg4 || arg5)
			return -EINVAL;
		error = arch_prctl_spec_ctrl_set(me, arg2, arg3);
		break;
	}
// snipped
	return error;
}
```

[https://elixir.bootlin.com/linux/v6.10.2/source/include/uapi/linux/prctl.h#L210](https://elixir.bootlin.com/linux/v6.10.2/source/include/uapi/linux/prctl.h#L210)

```c
/* Per task speculation control */
#define PR_GET_SPECULATION_CTRL		52
#define PR_SET_SPECULATION_CTRL		53
```

### [`arch_prctl_spec_ctrl_get()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L2338)

```c
int arch_prctl_spec_ctrl_get(struct task_struct *task, unsigned long which)
{
	switch (which) {
	case PR_SPEC_STORE_BYPASS:
		return ssb_prctl_get(task);
// snipped
	default:
		return -ENODEV;
	}
}
```

[https://elixir.bootlin.com/linux/v6.10.2/source/include/uapi/linux/prctl.h#L213](https://elixir.bootlin.com/linux/v6.10.2/source/include/uapi/linux/prctl.h#L213)

```c
/* Speculation control variants */
# define PR_SPEC_STORE_BYPASS		0
```

### [`ssb_prctl_get()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L2294)

Mitigation Selection で選択された緩和策に応じて、返る値が変わる。

- `SPEC_STORE_BYPASS_NONE` (SSB に対して何もしない)
    - `X86_BUG_SPEC_STORE_BYPASS` (SSB に脆弱): `PR_SPEC_ENABLE` (Speculation が有効 = 緩和策が無効)
    - SSB の影響を受けない: `PR_SPEC_NOT_AFFECTED`
- `SPEC_STORE_BYPASS_DISABLE` (常に SSBD を有効): `PR_SPEC_DISABLE` (Speculation が無効 = 緩和策が有効)
- `SPEC_STORE_BYPASS_SECCOMP` or `SPEC_STORE_BYPASS_PRCTL`
    - `PFA_SPEC_SSB_FORCE_DISABLE` (SSBD 有効 & 後から無効にできない): `PR_SPEC_PRCTL | PR_SPEC_FORCE_DISABLE`
    - `PFA_SPEC_SSB_NOEXEC` (`execve()` 実行時に SSBD 無効): `PR_SPEC_PRCTL | PR_SPEC_DISABLE_NOEXEC`
    - `PFA_SPEC_SSB_DISABLE` (SSBD 有効 & 後から無効化可能): `PR_SPEC_PRCTL | PR_SPEC_ENABLE`

```c
static int ssb_prctl_get(struct task_struct *task)
{
	switch (ssb_mode) {
	case SPEC_STORE_BYPASS_NONE:
		if (boot_cpu_has_bug(X86_BUG_SPEC_STORE_BYPASS))
			return PR_SPEC_ENABLE;
		return PR_SPEC_NOT_AFFECTED;
	case SPEC_STORE_BYPASS_DISABLE:
		return PR_SPEC_DISABLE;
	case SPEC_STORE_BYPASS_SECCOMP:
	case SPEC_STORE_BYPASS_PRCTL:
		if (task_spec_ssb_force_disable(task))
			return PR_SPEC_PRCTL | PR_SPEC_FORCE_DISABLE;
		if (task_spec_ssb_noexec(task))
			return PR_SPEC_PRCTL | PR_SPEC_DISABLE_NOEXEC;
		if (task_spec_ssb_disable(task))
			return PR_SPEC_PRCTL | PR_SPEC_DISABLE;
		return PR_SPEC_PRCTL | PR_SPEC_ENABLE;
	}
	BUG();
}
```

[https://elixir.bootlin.com/linux/v6.10.2/source/include/uapi/linux/prctl.h#L217](https://elixir.bootlin.com/linux/v6.10.2/source/include/uapi/linux/prctl.h#L217)

```c
/* Return and control values for PR_SET/GET_SPECULATION_CTRL */
# define PR_SPEC_NOT_AFFECTED		0
# define PR_SPEC_PRCTL			(1UL << 0)
# define PR_SPEC_ENABLE			(1UL << 1)
# define PR_SPEC_DISABLE		(1UL << 2)
# define PR_SPEC_FORCE_DISABLE		(1UL << 3)
# define PR_SPEC_DISABLE_NOEXEC		(1UL << 4)
```

[https://elixir.bootlin.com/linux/v6.10.2/source/Documentation/userspace-api/spec_ctrl.rst#L31](https://elixir.bootlin.com/linux/v6.10.2/source/Documentation/userspace-api/spec_ctrl.rst#L31)

```
==== ====================== ==================================================
Bit  Define                 Description
==== ====================== ==================================================
0    PR_SPEC_PRCTL          Mitigation can be controlled per task by
                            PR_SET_SPECULATION_CTRL.
1    PR_SPEC_ENABLE         The speculation feature is enabled, mitigation is
                            disabled.
2    PR_SPEC_DISABLE        The speculation feature is disabled, mitigation is
                            enabled.
3    PR_SPEC_FORCE_DISABLE  Same as PR_SPEC_DISABLE, but cannot be undone. A
                            subsequent prctl(..., PR_SPEC_ENABLE) will fail.
4    PR_SPEC_DISABLE_NOEXEC Same as PR_SPEC_DISABLE, but the state will be
                            cleared on :manpage:`execve(2)`.
==== ====================== ==================================================
```

### [`task_spec_ssb_force_disable()` / `task_spec_ssb_noexec()` / `task_spec_ssb_disable()`](https://elixir.bootlin.com/linux/v6.10.2/source/include/linux/sched.h#L1728)

```c
TASK_PFA_TEST(SPEC_SSB_DISABLE, spec_ssb_disable)
TASK_PFA_SET(SPEC_SSB_DISABLE, spec_ssb_disable)
TASK_PFA_CLEAR(SPEC_SSB_DISABLE, spec_ssb_disable)

TASK_PFA_TEST(SPEC_SSB_NOEXEC, spec_ssb_noexec)
TASK_PFA_SET(SPEC_SSB_NOEXEC, spec_ssb_noexec)
TASK_PFA_CLEAR(SPEC_SSB_NOEXEC, spec_ssb_noexec)

TASK_PFA_TEST(SPEC_SSB_FORCE_DISABLE, spec_ssb_force_disable)
TASK_PFA_SET(SPEC_SSB_FORCE_DISABLE, spec_ssb_force_disable)
```

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
```

```c
/* Per-process atomic flags. */
// snipped
#define PFA_SPEC_SSB_DISABLE		3	/* Speculative Store Bypass disabled */
#define PFA_SPEC_SSB_FORCE_DISABLE	4	/* Speculative Store Bypass force disabled*/
// snipped
#define PFA_SPEC_SSB_NOEXEC		7	/* Speculative Store Bypass clear on execve() */
```

### [`arch_prctl_spec_ctrl_set()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L2257)

```c
int arch_prctl_spec_ctrl_set(struct task_struct *task, unsigned long which,
			     unsigned long ctrl)
{
	switch (which) {
	case PR_SPEC_STORE_BYPASS:
		return ssb_prctl_set(task, ctrl);
// snipped
	default:
		return -ENODEV;
	}
}
```

### [`ssb_prctl_set()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L2154)

Mitigation Selection で選ばれた `ssb_mode` が `SPEC_STORE_BYPASS_PRCTL` でも `SPEC_STORE_BYPASS_SECCOMP` でもなければ、`prctl()` を通した制御はできないので `-ENXIO` を返す。

- `PR_SPEC_ENABLE`
    - `PFA_SPEC_SSB_FORCE_DISABLE` がすでに設定されている場合は `-EPERM` を返す。
    - `PFA_SPEC_SSB_DISABLE` と `PFA_SPEC_SSB_NOEXEC` をクリアする。
    - `TIF_SPEC_FORCE_UPDATE` フラグを立てる。
- `PR_SPEC_DISABLE`
    - `PFA_SPEC_SSB_DISABLE` をセットする。
    - `PFA_SPEC_SSB_NOEXEC` をクリアする。
    - `TIF_SPEC_FORCE_UPDATE` フラグを立てる。
- `PR_SPEC_FORCE_DISABLE`
    - `PFA_SPEC_SSB_DISABLE` と `PFA_SPEC_SSB_FORCE_DISABLE` をセットする。
    - `PFA_SPEC_SSB_NOEXEC` をクリアする。
    - `TIF_SPEC_FORCE_UPDATE` フラグを立てる。
- `PR_SPEC_DISABLE_NOEXEC`
    - `PFA_SPEC_SSB_FORCE_DISABLE` がすでに設定されちる場合は `-EPERM` を返す。
    - `PFA_SPEC_SSB_DISABLE` と `PFA_SPEC_SSB_NOEXEC` をセットする。
    - `TIF_SPEC_FORCE_UPDATE` フラグを立てる。

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

### [`task_update_spec_tif()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L2119)

`prctl()` を呼んだ場合は、常に `tsk == current` であり、すでに Speculation Control MSR を更新する。
seccomp の場合は、次にスケジュールされた時に適用される。

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

### [`TIF_SPEC_FORCE_UPDATE`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/thread_info.h#L104)

コンテキストスイッチ時に Speculation Control MSR を更新するフラグである。

```c
/*
 * thread information flags
 * - these are process state flags that various assembly files
 *   may need to access
 */
// snipped
#define TIF_SPEC_FORCE_UPDATE	23	/* Force speculation MSR update in context switch */
// snipped
#define _TIF_SPEC_FORCE_UPDATE	(1 << TIF_SPEC_FORCE_UPDATE)
```

### [`set_tsk_thread_flag()`](https://elixir.bootlin.com/linux/v6.10.2/source/include/linux/sched.h#L1928)

```c
/*
 * Set thread flags in other task's structures.
 * See asm/thread_info.h for TIF_xxxx flags available:
 */
static inline void set_tsk_thread_flag(struct task_struct *tsk, int flag)
{
	set_ti_thread_flag(task_thread_info(tsk), flag);
}
```

### [`set_ti_thread_flag()`](https://elixir.bootlin.com/linux/v6.10.2/source/include/linux/thread_info.h#L87)

```c
static inline void set_ti_thread_flag(struct thread_info *ti, int flag)
{
	set_bit(flag, (unsigned long *)&ti->flags);
}
```

### [`struct thread_info`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/thread_info.h#L62)

```c
struct thread_info {
	unsigned long		flags;		/* low level flags */
	unsigned long		syscall_work;	/* SYSCALL_WORK_ flags */
	u32			status;		/* thread synchronous flags */
#ifdef CONFIG_SMP
	u32			cpu;		/* current CPU */
#endif
};
```

### [`speculation_ctrl_update_current()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/process.c#L674)

`current` にアクセスするので、preemption を無効にしている。

```c
/* Called from seccomp/prctl update */
void speculation_ctrl_update_current(void)
{
	preempt_disable();
	speculation_ctrl_update(speculation_ctrl_update_tif(current));
	preempt_enable();
}
```

### [`speculation_ctrl_update_tif()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/process.c#L646)

`TIF_SPEC_FORCE_UPDATE` がセットされている場合、`PFA_SPEC_SSB_DISABLE` がセットされているかを確認して、それに応じて `TIF_SSBD` をセットする。

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

### [`TIF_SSBD`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/thread_info.h#L91)

```c
/*
 * thread information flags
 * - these are process state flags that various assembly files
 *   may need to access
 */
// snipped
#define TIF_SSBD		5	/* Speculative store bypass disable */

// snipped
#define _TIF_SSBD		(1 << TIF_SSBD)
```

### [`speculation_ctrl_update()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/process.c#L663)

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

### [`__speculation_ctrl_update()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/process.c#L613)

強制アップデート時とコンテキストスイッチ時に呼び出される Speculation Control MSR を更新する関数。
ハードウェアで利用可能な SSBD の種類に応じて、別々の関数をディスパッチする。
ここでは最も一般的な `X86_FEATURE_SPEC_CTRL_SSBD` または `X86_FEATURE_SPEC_AMD_SSBD` のケースを深ぼる。

最初に `tif_diff = tifp ^ tifn` を計算しており、`speculation_ctrl_update()` から与えられた値より `tif_diff = ~tif ^ tif` となり、全てのビットが立った状態になる。

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

// snipped
	if (updmsr)
		update_spec_ctrl_cond(msr);
}
```

### [`ssbd_tif_to_spec_ctrl()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/include/asm/spec-ctrl.h#L51)

TIF 内の SSBD のビットポジションを、Speculation Control MSR 内の SSBD のビットポジションへ変換。

```c
static inline u64 ssbd_tif_to_spec_ctrl(u64 tifn)
{
	BUILD_BUG_ON(TIF_SSBD < SPEC_CTRL_SSBD_SHIFT);
	return (tifn & _TIF_SSBD) >> (TIF_SSBD - SPEC_CTRL_SSBD_SHIFT);
}
```

### [`update_spec_ctrl_cond()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L81)

現在の値を同じであればスキップする。
`x86_spec_ctrl_current` に値をセットし、`X86_FEATURE_KERNEL_IBRS` がセットされている場合は、ユーザー空間に戻るまで遅延される模様。

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

### [`x86_spec_ctrl_current`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L57)

```c
/* The current value of the SPEC_CTRL MSR with task-specific bits set */
DEFINE_PER_CPU(u64, x86_spec_ctrl_current);
EXPORT_PER_CPU_SYMBOL_GPL(x86_spec_ctrl_current);
```


## `/proc/$pid/status`

### [`tid_base_stuff[]`](https://elixir.bootlin.com/linux/v6.10.2/source/fs/proc/base.c#L3610)

```c
/*
 * Tasks
 */
static const struct pid_entry tid_base_stuff[] = {
// snipped
	ONE("status",    S_IRUGO, proc_pid_status),
// snipped
};
```

### [`tgid_base_stuff[]`](https://elixir.bootlin.com/linux/v6.10.2/source/fs/proc/base.c#L3259)

```c
static const struct pid_entry tgid_base_stuff[] = {
// snipped
	ONE("status",     S_IRUGO, proc_pid_status),
// snipped
};
```

### [`proc_pid_status()`](https://elixir.bootlin.com/linux/v6.10.2/source/fs/proc/array.c#L439)

```c
int proc_pid_status(struct seq_file *m, struct pid_namespace *ns,
			struct pid *pid, struct task_struct *task)
{
// snipped
	task_seccomp(m, task);
// snipped
	return 0;
}
```

### [`task_seccomp()`](https://elixir.bootlin.com/linux/v6.10.2/source/fs/proc/array.c#L333)

```c
static inline void task_seccomp(struct seq_file *m, struct task_struct *p)
{
// snipped
	seq_puts(m, "\nSpeculation_Store_Bypass:\t");
	switch (arch_prctl_spec_ctrl_get(p, PR_SPEC_STORE_BYPASS)) {
	case -EINVAL:
		seq_puts(m, "unknown");
		break;
	case PR_SPEC_NOT_AFFECTED:
		seq_puts(m, "not vulnerable");
		break;
	case PR_SPEC_PRCTL | PR_SPEC_FORCE_DISABLE:
		seq_puts(m, "thread force mitigated");
		break;
	case PR_SPEC_PRCTL | PR_SPEC_DISABLE:
		seq_puts(m, "thread mitigated");
		break;
	case PR_SPEC_PRCTL | PR_SPEC_ENABLE:
		seq_puts(m, "thread vulnerable");
		break;
	case PR_SPEC_DISABLE:
		seq_puts(m, "globally mitigated");
		break;
	default:
		seq_puts(m, "vulnerable");
		break;
	}
// snipped
}
```


## seccomp

### [`seccomp()`](https://elixir.bootlin.com/linux/v6.10.2/source/kernel/seccomp.c#L2074)

```c
SYSCALL_DEFINE3(seccomp, unsigned int, op, unsigned int, flags,
			 void __user *, uargs)
{
	return do_seccomp(op, flags, uargs);
}
```

### [`do_seccomp()`](https://elixir.bootlin.com/linux/v6.10.2/source/kernel/seccomp.c#L2046)

```c
/* Common entry point for both prctl and syscall. */
static long do_seccomp(unsigned int op, unsigned int flags,
		       void __user *uargs)
{
	switch (op) {
// snipped
	case SECCOMP_SET_MODE_FILTER:
		return seccomp_set_mode_filter(flags, uargs);
// snipped
	default:
		return -EINVAL;
	}
}
```

### [`seccomp_assign_mode()`](https://elixir.bootlin.com/linux/v6.10.2/source/kernel/seccomp.c#L449)

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

### [`SECCOMP_FILTER_FLAG_SPEC_ALLOW`](https://elixir.bootlin.com/linux/v6.10.2/source/include/uapi/linux/seccomp.h#L23)

```c
/* Valid flags for SECCOMP_SET_MODE_FILTER */
// snipped
#define SECCOMP_FILTER_FLAG_SPEC_ALLOW		(1UL << 2)
```

### [`arch_seccomp_spec_mitigate()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/kernel/cpu/bugs.c#L2273)

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



# arm64 (※ まだ未完成です)

以下のパッチセットが最初に取り込まれた SSB への緩和策のパッチセット

https://lore.kernel.org/all/20180522150648.28297-1-marc.zyngier@arm.com/

- [arm/arm64: smccc: Add SMCCC-specific return codes](https://github.com/torvalds/linux/commit/eff0e9e1078ea7dc1d794dc50e31baef984c46d7)
- [arm64: Call ARCH_WORKAROUND_2 on transitions between EL0 and EL1](https://github.com/torvalds/linux/commit/8e2906245f1e3b0d027169d9f2e55ce0548cb96e)
- [arm64: Add per-cpu infrastructure to call ARCH_WORKAROUND_2](https://github.com/torvalds/linux/commit/5cf9ce6e5ea50f805c6188c04ed0daaec7b6887d)
- [arm64: Add ARCH_WORKAROUND_2 probing](https://github.com/torvalds/linux/commit/a725e3dda1813ed306734823ac4c65ca04e38500)
- [arm64: Add 'ssbd' command-line option](https://github.com/torvalds/linux/commit/a43ae4dfe56a01f5b98ba0cb2f784b6a43bafcc6)
- [arm64: ssbd: Add global mitigation state accessor](https://github.com/torvalds/linux/commit/c32e1736ca03904c03de0e4459a673be194f56fd)
- [arm64: ssbd: Skip apply_ssbd if not using dynamic mitigation](https://github.com/torvalds/linux/commit/986372c4367f46b34a3c0f6918d7fb95cbdf39d6)
- [arm64: ssbd: Restore mitigation status on CPU resume](https://github.com/torvalds/linux/commit/647d0519b53f440a55df163de21c52a8205431cc)
- [arm64: ssbd: Introduce thread flag to control userspace mitigation](https://github.com/torvalds/linux/commit/9dd9614f5476687abbff8d4b12cd08ae70d7c2ad)
- [arm64: ssbd: Add prctl interface for per-thread mitigation](https://github.com/torvalds/linux/commit/9cdc0108baa8ef87c76ed834619886a46bd70cbe)
- [arm64: KVM: Add HYP per-cpu accessors](https://github.com/torvalds/linux/commit/85478bab409171de501b719971fd25a3d5d639f9)
- [arm64: KVM: Add ARCH_WORKAROUND_2 support for guests](https://github.com/torvalds/linux/commit/55e3748e8902ff641e334226bdcb432f9a5d78d3)
- [arm64: KVM: Handle guest's ARCH_WORKAROUND_2 requests](https://github.com/torvalds/linux/commit/b4f18c063a13dfb33e3a63fe1844823e19c2265e)
- [arm64: KVM: Add ARCH_WORKAROUND_2 discovery through ARCH_FEATURES_FUNC_ID](https://github.com/torvalds/linux/commit/5d81f7dc9bca4f4963092433e27b508cbe524a32)


## Terminology

- SMCCC: SMC Calling Convention
- SMC: Secure Monitor Call
- HVC: Hypervisor Call
- PSCI: Power State Coordination Interface

[Firmware interfaces for mitigating cache speculation vulnerabilities](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Security%20update%2021%20May%2018/ARM%20DEN%200070A%20Firmware%20interfaces%20for%20mitigating%20cache%20speculation%20vulnerabilities.pdf?la=en&revision=2a836de0-4661-44ba-a40b-1e73ca2fcb29)

```
2.2.5 SMCCC_ARCH_WORKAROUND_2

+-------------+-----------------------------------------------------------------------------------------------+
| Dependency  | OPTIONAL from SMCCC v1.1                                                                      |
|             | NOT SUPPORTED in SMCCC v1.0                                                                   |
+-------------+-----------------------------------------------------------------------------------------------+
| Description | Enable or disable the mitigation for CVE-2018-3639 on the calling PE                          |
+-------------+--------------------+--------------------------------------------------------------------------+
| Parameters  | uint32 Function ID | 0x8000 7FFF                                                              |
|             +--------------------+--------------------------------------------------------------------------+
|             | uint32 enable      | A non-zero value indicates that the mitigation for CVE-2018-3639 must be |
|             |                    | enabled. A value of zero indicates that it must be disabled.             |
+-------------+--------------------+--------------------------------------------------------------------------+
| Return      | void               | This function has no return value.                                       |
+-------------+--------------------+--------------------------------------------------------------------------+
```

## Bug Detection & SSBD Feature Detection

### [`struct arm64_cpu_capabilities`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/include/asm/cpufeature.h#L334)

```c
/*
 * CPU capabilities:
 *
 * We use arm64_cpu_capabilities to represent system features, errata work
 * arounds (both used internally by kernel and tracked in system_cpucaps) and
 * ELF HWCAPs (which are exposed to user).
 *
 * To support systems with heterogeneous CPUs, we need to make sure that we
 * detect the capabilities correctly on the system and take appropriate
 * measures to ensure there are no incompatibilities.
 *
 * This comment tries to explain how we treat the capabilities.
 * Each capability has the following list of attributes :
 *
 * 1) Scope of Detection : The system detects a given capability by
 *    performing some checks at runtime. This could be, e.g, checking the
 *    value of a field in CPU ID feature register or checking the cpu
 *    model. The capability provides a call back ( @matches() ) to
 *    perform the check. Scope defines how the checks should be performed.
 *    There are three cases:
 *
 *     a) SCOPE_LOCAL_CPU: check all the CPUs and "detect" if at least one
 *        matches. This implies, we have to run the check on all the
 *        booting CPUs, until the system decides that state of the
 *        capability is finalised. (See section 2 below)
 *		Or
 *     b) SCOPE_SYSTEM: check all the CPUs and "detect" if all the CPUs
 *        matches. This implies, we run the check only once, when the
 *        system decides to finalise the state of the capability. If the
 *        capability relies on a field in one of the CPU ID feature
 *        registers, we use the sanitised value of the register from the
 *        CPU feature infrastructure to make the decision.
 *		Or
 *     c) SCOPE_BOOT_CPU: Check only on the primary boot CPU to detect the
 *        feature. This category is for features that are "finalised"
 *        (or used) by the kernel very early even before the SMP cpus
 *        are brought up.
 *
 *    The process of detection is usually denoted by "update" capability
 *    state in the code.
 *
 * 2) Finalise the state : The kernel should finalise the state of a
 *    capability at some point during its execution and take necessary
 *    actions if any. Usually, this is done, after all the boot-time
 *    enabled CPUs are brought up by the kernel, so that it can make
 *    better decision based on the available set of CPUs. However, there
 *    are some special cases, where the action is taken during the early
 *    boot by the primary boot CPU. (e.g, running the kernel at EL2 with
 *    Virtualisation Host Extensions). The kernel usually disallows any
 *    changes to the state of a capability once it finalises the capability
 *    and takes any action, as it may be impossible to execute the actions
 *    safely. A CPU brought up after a capability is "finalised" is
 *    referred to as "Late CPU" w.r.t the capability. e.g, all secondary
 *    CPUs are treated "late CPUs" for capabilities determined by the boot
 *    CPU.
 *
 *    At the moment there are two passes of finalising the capabilities.
 *      a) Boot CPU scope capabilities - Finalised by primary boot CPU via
 *         setup_boot_cpu_capabilities().
 *      b) Everything except (a) - Run via setup_system_capabilities().
 *
 * 3) Verification: When a CPU is brought online (e.g, by user or by the
 *    kernel), the kernel should make sure that it is safe to use the CPU,
 *    by verifying that the CPU is compliant with the state of the
 *    capabilities finalised already. This happens via :
 *
 *	secondary_start_kernel()-> check_local_cpu_capabilities()
 *
 *    As explained in (2) above, capabilities could be finalised at
 *    different points in the execution. Each newly booted CPU is verified
 *    against the capabilities that have been finalised by the time it
 *    boots.
 *
 *	a) SCOPE_BOOT_CPU : All CPUs are verified against the capability
 *	except for the primary boot CPU.
 *
 *	b) SCOPE_LOCAL_CPU, SCOPE_SYSTEM: All CPUs hotplugged on by the
 *	user after the kernel boot are verified against the capability.
 *
 *    If there is a conflict, the kernel takes an action, based on the
 *    severity (e.g, a CPU could be prevented from booting or cause a
 *    kernel panic). The CPU is allowed to "affect" the state of the
 *    capability, if it has not been finalised already. See section 5
 *    for more details on conflicts.
 *
 * 4) Action: As mentioned in (2), the kernel can take an action for each
 *    detected capability, on all CPUs on the system. Appropriate actions
 *    include, turning on an architectural feature, modifying the control
 *    registers (e.g, SCTLR, TCR etc.) or patching the kernel via
 *    alternatives. The kernel patching is batched and performed at later
 *    point. The actions are always initiated only after the capability
 *    is finalised. This is usally denoted by "enabling" the capability.
 *    The actions are initiated as follows :
 *	a) Action is triggered on all online CPUs, after the capability is
 *	finalised, invoked within the stop_machine() context from
 *	enable_cpu_capabilitie().
 *
 *	b) Any late CPU, brought up after (1), the action is triggered via:
 *
 *	  check_local_cpu_capabilities() -> verify_local_cpu_capabilities()
 *
 * 5) Conflicts: Based on the state of the capability on a late CPU vs.
 *    the system state, we could have the following combinations :
 *
 *		x-----------------------------x
 *		| Type  | System   | Late CPU |
 *		|-----------------------------|
 *		|  a    |   y      |    n     |
 *		|-----------------------------|
 *		|  b    |   n      |    y     |
 *		x-----------------------------x
 *
 *     Two separate flag bits are defined to indicate whether each kind of
 *     conflict can be allowed:
 *		ARM64_CPUCAP_OPTIONAL_FOR_LATE_CPU - Case(a) is allowed
 *		ARM64_CPUCAP_PERMITTED_FOR_LATE_CPU - Case(b) is allowed
 *
 *     Case (a) is not permitted for a capability that the system requires
 *     all CPUs to have in order for the capability to be enabled. This is
 *     typical for capabilities that represent enhanced functionality.
 *
 *     Case (b) is not permitted for a capability that must be enabled
 *     during boot if any CPU in the system requires it in order to run
 *     safely. This is typical for erratum work arounds that cannot be
 *     enabled after the corresponding capability is finalised.
 *
 *     In some non-typical cases either both (a) and (b), or neither,
 *     should be permitted. This can be described by including neither
 *     or both flags in the capability's type field.
 *
 *     In case of a conflict, the CPU is prevented from booting. If the
 *     ARM64_CPUCAP_PANIC_ON_CONFLICT flag is specified for the capability,
 *     then a kernel panic is triggered.
 */
```
```c
struct arm64_cpu_capabilities {
	const char *desc;
	u16 capability;
	u16 type;
	bool (*matches)(const struct arm64_cpu_capabilities *caps, int scope);
	/*
	 * Take the appropriate actions to configure this capability
	 * for this CPU. If the capability is detected by the kernel
	 * this will be called on all the CPUs in the system,
	 * including the hotplugged CPUs, regardless of whether the
	 * capability is available on that specific CPU. This is
	 * useful for some capabilities (e.g, working around CPU
	 * errata), where all the CPUs must take some action (e.g,
	 * changing system control/configuration). Thus, if an action
	 * is required only if the CPU has the capability, then the
	 * routine must check it before taking any action.
	 */
	void (*cpu_enable)(const struct arm64_cpu_capabilities *cap);
	union {
		struct {	/* To be used for erratum handling only */
			struct midr_range midr_range;
			const struct arm64_midr_revidr {
				u32 midr_rv;		/* revision/variant */
				u32 revidr_mask;
			} * const fixed_revs;
		};

		const struct midr_range *midr_range_list;
		struct {	/* Feature register checking */
			u32 sys_reg;
			u8 field_pos;
			u8 field_width;
			u8 min_field_value;
			u8 max_field_value;
			u8 hwcap_type;
			bool sign;
			unsigned long hwcap;
		};
	};

	/*
	 * An optional list of "matches/cpu_enable" pair for the same
	 * "capability" of the same "type" as described by the parent.
	 * Only matches(), cpu_enable() and fields relevant to these
	 * methods are significant in the list. The cpu_enable is
	 * invoked only if the corresponding entry "matches()".
	 * However, if a cpu_enable() method is associated
	 * with multiple matches(), care should be taken that either
	 * the match criteria are mutually exclusive, or that the
	 * method is robust against being called multiple times.
	 */
	const struct arm64_cpu_capabilities *match_list;
	const struct cpumask *cpus;
};
```

### [`arm64_errata[]`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/kernel/cpu_errata.c#L564)

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

[https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/include/asm/cpufeature.h#L286](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/include/asm/cpufeature.h#L286)

```c
/*
 * CPU errata workarounds that need to be enabled at boot time if one or
 * more CPUs in the system requires it. When one of these capabilities
 * has been enabled, it is safe to allow any CPU to boot that doesn't
 * require the workaround. However, it is not safe if a "late" CPU
 * requires a workaround and the system hasn't enabled it already.
 */
#define ARM64_CPUCAP_LOCAL_CPU_ERRATUM		\
	(ARM64_CPUCAP_SCOPE_LOCAL_CPU | ARM64_CPUCAP_OPTIONAL_FOR_LATE_CPU)
```

### [`arm64_features[]`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/kernel/cpufeature.c#L2592)

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

[https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/include/asm/cpufeature.h#L295](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/include/asm/cpufeature.h#L295)

```c
/*
 * CPU feature detected at boot time based on system-wide value of a
 * feature. It is safe for a late CPU to have this feature even though
 * the system hasn't enabled it, although the feature will not be used
 * by Linux in this case. If the system has enabled this feature already,
 * then every late CPU must have it.
 */
#define ARM64_CPUCAP_SYSTEM_FEATURE	\
	(ARM64_CPUCAP_SCOPE_SYSTEM | ARM64_CPUCAP_PERMITTED_FOR_LATE_CPU)
```

### [`has_spectre_v4()`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/kernel/proton-pack.c#L511)

まずはハードウェアでの緩和策があるかを確認して、その後にファームウェアでの緩和策があるかを確認する。
`SPECTRE_UNAFFECTED` でない場合、spectre_v4 に影響を受けるものとして判定される。

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

### [`spectre_v4_get_cpu_hw_mitigation_state()`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/kernel/proton-pack.c#L466)

```c
static enum mitigation_state spectre_v4_get_cpu_hw_mitigation_state(void)
{
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

	/* CPU features are detected first */
	if (this_cpu_has_cap(ARM64_SSBS))
		return SPECTRE_MITIGATED;

	return SPECTRE_VULNERABLE;
}
```

### [`this_cpu_has_cap()`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/kernel/cpufeature.c#L3423)
```c
bool this_cpu_has_cap(unsigned int n)
{
	if (!WARN_ON(preemptible()) && n < ARM64_NCAPS) {
		const struct arm64_cpu_capabilities *cap = cpucap_ptrs[n];

		if (cap)
			return cap->matches(cap, SCOPE_LOCAL_CPU);
	}

	return false;
}
EXPORT_SYMBOL_GPL(this_cpu_has_cap);
```

### [`has_cpuid_feature()`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/kernel/cpufeature.c#L1601)

```c
static bool
has_cpuid_feature(const struct arm64_cpu_capabilities *entry, int scope)
{
	u64 val = read_scoped_sysreg(entry, scope);
	return feature_matches(val, entry);
}
```

### [`spectre_v4_get_cpu_fw_mitigation_state()`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/kernel/proton-pack.c#L488)

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

[https://elixir.bootlin.com/linux/v6.10.2/source/include/linux/arm-smccc.h#L93](https://elixir.bootlin.com/linux/v6.10.2/source/include/linux/arm-smccc.h#L93)

```c
#define ARM_SMCCC_ARCH_WORKAROUND_2					\
	ARM_SMCCC_CALL_VAL(ARM_SMCCC_FAST_CALL,				\
			   ARM_SMCCC_SMC_32,				\
			   0, 0x7fff)
```

[https://elixir.bootlin.com/linux/v6.10/source/include/linux/arm-smccc.h#L192](https://elixir.bootlin.com/linux/v6.10/source/include/linux/arm-smccc.h#L192)

```c
/*
 * Return codes defined in ARM DEN 0070A
 * ARM DEN 0070A is now merged/consolidated into ARM DEN 0028 C
 */
#define SMCCC_RET_SUCCESS			0
#define SMCCC_RET_NOT_SUPPORTED			-1
#define SMCCC_RET_NOT_REQUIRED			-2
#define SMCCC_RET_INVALID_PARAMETER		-3
```

[https://elixir.bootlin.com/linux/v6.10/source/include/linux/arm-smccc.h#L127](https://elixir.bootlin.com/linux/v6.10/source/include/linux/arm-smccc.h#L127)

```c
#define SMCCC_ARCH_WORKAROUND_RET_UNAFFECTED	1
```

### [`enum mitigation_state`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/include/asm/spectre.h#L23)

```c
/* Watch out, ordering is important here. */
enum mitigation_state {
	SPECTRE_UNAFFECTED,
	SPECTRE_MITIGATED,
	SPECTRE_VULNERABLE,
};
```


## Mitigation Selection

https://elixir.bootlin.com/linux/v6.10.2/source/Documentation/admin-guide/kernel-parameters.txt#L6398

```
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

### [`enum spectre_v4_policy`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L378)

```c
enum spectre_v4_policy {
	SPECTRE_V4_POLICY_MITIGATION_DYNAMIC,
	SPECTRE_V4_POLICY_MITIGATION_ENABLED,
	SPECTRE_V4_POLICY_MITIGATION_DISABLED,
};

static enum spectre_v4_policy __read_mostly __spectre_v4_policy;
```

### [`spectre_v4_params[]`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L386)

```c
static const struct spectre_v4_param {
	const char		*str;
	enum spectre_v4_policy	policy;
} spectre_v4_params[] = {
	{ "force-on",	SPECTRE_V4_POLICY_MITIGATION_ENABLED, },
	{ "force-off",	SPECTRE_V4_POLICY_MITIGATION_DISABLED, },
	{ "kernel",	SPECTRE_V4_POLICY_MITIGATION_DYNAMIC, },
};
```

### [`parse_spectre_v4_param()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L394)

```c
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

### [`spectre_v4_enable_mitigation()`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/kernel/proton-pack.c#L643)

`has_spectre_v4()` と同様に、まずはハードウェアの緩和策の有効化を試みて、その後にファームウェアの緩和策の有効化を試みる。
最後に `spectre_v4_state` を更新する。

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

### [`spectre_v4_enable_hw_mitigation()`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/kernel/proton-pack.c#L541)

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

	if (spectre_v4_mitigations_off()) {
		sysreg_clear_set(sctlr_el1, 0, SCTLR_ELx_DSSBS);
		set_pstate_ssbs(1);
		return SPECTRE_VULNERABLE;
	}

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
	if (IS_ENABLED(CONFIG_ARM64_WORKAROUND_SPECULATIVE_SSBS))
		spec_bar();

	return SPECTRE_MITIGATED;
}
```

### [`spectre_v4_enable_fw_mitigation()`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/kernel/proton-pack.c#L622)

```c
static enum mitigation_state spectre_v4_enable_fw_mitigation(void)
{
	enum mitigation_state state;

	state = spectre_v4_get_cpu_fw_mitigation_state();
	if (state != SPECTRE_MITIGATED)
		return state;

	if (spectre_v4_mitigations_off()) {
		arm_smccc_1_1_invoke(ARM_SMCCC_ARCH_WORKAROUND_2, false, NULL);
		return SPECTRE_VULNERABLE;
	}

	arm_smccc_1_1_invoke(ARM_SMCCC_ARCH_WORKAROUND_2, true, NULL);

	if (spectre_v4_mitigations_dynamic())
		__this_cpu_write(arm64_ssbd_callback_required, 1);

	return SPECTRE_MITIGATED;
}
```

### [`spectre_v4_mitigations_off()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L422)

```c
static bool spectre_v4_mitigations_off(void)
{
	bool ret = cpu_mitigations_off() ||
		   __spectre_v4_policy == SPECTRE_V4_POLICY_MITIGATION_DISABLED;

	if (ret)
		pr_info_once("spectre-v4 mitigation disabled by command-line option\n");

	return ret;
}
```

### [`spectre_v4_mitigations_dynamic()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L434)

```c
/* Do we need to toggle the mitigation state on entry to/exit from the kernel? */
static bool spectre_v4_mitigations_dynamic(void)
{
	return !spectre_v4_mitigations_off() &&
	       __spectre_v4_policy == SPECTRE_V4_POLICY_MITIGATION_DYNAMIC;
}
```

### [`spectre_v4_mitigations_on()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L440)

```c
static bool spectre_v4_mitigations_on(void)
{
	return !spectre_v4_mitigations_off() &&
	       __spectre_v4_policy == SPECTRE_V4_POLICY_MITIGATION_ENABLED;
}
```

### [`spectre_v4_state`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/kernel/proton-pack.c#L373)

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


## sysfs

### [`spec_store_bypass`](https://elixir.bootlin.com/linux/v6.10/source/drivers/base/cpu.c#L596)

```c
static DEVICE_ATTR(spec_store_bypass, 0444, cpu_show_spec_store_bypass, NULL);
```

### [`cpu_show_spec_store_bypass()`](https://elixir.bootlin.com/linux/v6.10/source/arch/arm64/kernel/proton-pack.c#L446)

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


## prctl

### [`prctl()`](https://elixir.bootlin.com/linux/v6.10.2/source/kernel/sys.c#L2457)

アーキテクチャ非依存なので、x86 で見たように `arch_prctl_spec_ctrl_get()` と `arch_prctl_spec_ctrl_set()` を呼び出す。

### [`arch_prctl_spec_ctrl_get()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L807)

```c
int arch_prctl_spec_ctrl_get(struct task_struct *task, unsigned long which)
{
	switch (which) {
	case PR_SPEC_STORE_BYPASS:
		return ssbd_prctl_get(task);
	default:
		return -ENODEV;
	}
}
```

### [`ssbd_prctl_get()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L774)

```c
static int ssbd_prctl_get(struct task_struct *task)
{
	switch (spectre_v4_state) {
	case SPECTRE_UNAFFECTED:
		return PR_SPEC_NOT_AFFECTED;
	case SPECTRE_MITIGATED:
		if (spectre_v4_mitigations_on())
			return PR_SPEC_NOT_AFFECTED;

		if (spectre_v4_mitigations_dynamic())
			break;

		/* Mitigations are disabled, so we're vulnerable. */
		fallthrough;
	case SPECTRE_VULNERABLE:
		fallthrough;
	default:
		return PR_SPEC_ENABLE;
	}

	/* Check the mitigation state for this task */
	if (task_spec_ssb_force_disable(task))
		return PR_SPEC_PRCTL | PR_SPEC_FORCE_DISABLE;

	if (task_spec_ssb_noexec(task))
		return PR_SPEC_PRCTL | PR_SPEC_DISABLE_NOEXEC;

	if (task_spec_ssb_disable(task))
		return PR_SPEC_PRCTL | PR_SPEC_DISABLE;

	return PR_SPEC_PRCTL | PR_SPEC_ENABLE;
}
```

### [`arch_prctl_spec_ctrl_set()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L763)

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

### [`ssbd_prctl_set()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L700)

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

### [`ssbd_prctl_disable_mitigation()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L693)

```c
static void ssbd_prctl_disable_mitigation(struct task_struct *task)
{
	task_clear_spec_ssb_noexec(task);
	task_clear_spec_ssb_disable(task);
	clear_tsk_thread_flag(task, TIF_SSBD);
}
```

### [`ssbd_prctl_enable_mitigation()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L686)

```c
static void ssbd_prctl_enable_mitigation(struct task_struct *task)
{
	task_clear_spec_ssb_noexec(task);
	task_set_spec_ssb_disable(task);
	set_tsk_thread_flag(task, TIF_SSBD);
}
```

### [`spectre_v4_enable_task_mitigation()`](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/proton-pack.c#L666)

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


## `/proc/$pid/status`

アーキテクチャ非依存なので x86 と同じ。


## seccomp

seccomp は x86 でしか使えない緩和策である。

### [`arch_seccomp_spec_mitigate()`](https://elixir.bootlin.com/linux/v6.10.2/source/kernel/seccomp.c#L447)

```c
void __weak arch_seccomp_spec_mitigate(struct task_struct *task) { }
```


[ARM_SMCCC_ARCH_WORKAROUND_2](https://elixir.bootlin.com/linux/v6.10.2/source/include/linux/arm-smccc.h#L93)

```c
#define ARM_SMCCC_ARCH_WORKAROUND_2					\
	ARM_SMCCC_CALL_VAL(ARM_SMCCC_FAST_CALL,				\
			   ARM_SMCCC_SMC_32,				\
			   0, 0x7fff)
```

[apply_ssbd](https://elixir.bootlin.com/linux/v6.10.2/source/arch/arm64/kernel/entry.S#L115)

```c
	/*
	 * This macro corrupts x0-x3. It is the caller's duty  to save/restore
	 * them if required.
	 */
	.macro	apply_ssbd, state, tmp1, tmp2
alternative_cb	ARM64_ALWAYS_SYSTEM, spectre_v4_patch_fw_mitigation_enable
	b	.L__asm_ssbd_skip\@		// Patched to NOP
alternative_cb_end
	ldr_this_cpu	\tmp2, arm64_ssbd_callback_required, \tmp1
	cbz	\tmp2,	.L__asm_ssbd_skip\@
	ldr	\tmp2, [tsk, #TSK_TI_FLAGS]
	tbnz	\tmp2, #TIF_SSBD, .L__asm_ssbd_skip\@
	mov	w0, #ARM_SMCCC_ARCH_WORKAROUND_2
	mov	w1, #\state
alternative_cb	ARM64_ALWAYS_SYSTEM, smccc_patch_fw_mitigation_conduit
	nop					// Patched to SMC/HVC #0
alternative_cb_end
.L__asm_ssbd_skip\@:
	.endm
```