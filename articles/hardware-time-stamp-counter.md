---
title: 【Hardware】Time-Stamp Counter (TSC)
emoji: "🛠️"
type: "tech"
topics: ["hardware", "tsc", "timer"]
published: true
---



# Time-Stamp Counter (TSC)

TSC は、IA32_TIME_STAMP_COUNTER MSR (アドレス: 10H) で提供される 64-bit カウンタである。
プロセッサのリセット時に 0 に設定され、プロセッサが HLT 命令や STPCLK# ピンにより Halt されてもカウントアップし続けるカウンタである。

CPUID.01H:EDX.TSC[bit 4] = 1 のプロセッサで、TSC が利用可能である。

RDTSC 命令を使うことで、TSC の値を読むことができ、単調増加する値が返ってくることが保証されている。
RDTSC 命令は、通常どの特権レベルのソフトウェアからも実行することができる。
ただし、CR4.TSD[bit 2] = 0 に設定することで、特権レベルが 0 のソフトウェアのみが RDTSC 命令を実行できないようにすることができる。
RDTSC 命令はシリアライズされず、カウンタを読み取る前にそれ以前の命令が全て実行済みである必要はなく、後続の命令が RDTSC 命令が完了する前に開始されうる。

他の MSR と同様に RDMSR 命令および WRMSR 命令を使って、IA32_TIME_STAMP_COUNTER MSR にアクセスすることができる。


## Invariant TSC

新しいプロセッサでは Invariant TSC がサポートされ、CPUID.80000007H:EDX[8] = 1 のプロセッサで利用可能である。
Invariant TSC では、ACPI P-、C-、T- 全ての状態で、一定のレートで TSC がカウントアップされる。
これにより、ACPI Timer や HPET Timer の代わりに TSC を Wall Clock Timer サービスのために利用することができる。


## IA32_TSC_AUX MSR / RDTSCP

IA32_TSC_AUX MSR (アドレス: C000_0103H) は、特権ソフトウェアによってシグネチャ (例えば Logical Processor ID) によって初期化された 32-bit のフィールドを提供する MSR であり、IA32_TIME_STAMP_COUNTER MSR と組み合わせて使うようにデザインされている。

RDTSCP 命令によって、IA32_TIME_STAMP_COUNTER MSR の 64-bit カウンタ値および IA32_TSC_AUX MSR の 32-bit シグネチャ値を、アトミックに読み込むことができる。
IA32_TIME_STAMP_COUNTER MSR の値は EDX:EAX に読み込まれ、IA32_TSC_AUX MSR の値は ECX に読み込まれる。

CPUID.80000001H:EDX[27] = 1 のプロセッサで利用可能である。


## Time-Stamp Counter Adjustment

WRMSR 命令を用いて IA32_TIMESTAMP_COUNTER MSR に書き込みを行うことで、TSC の値を変更することができる。
そのような書き込みは、WRMSR 命令を実行したロジカルプロセッサのみに適用されるので、複数のロジカルプロセッサの TSC の値を同期させたいソフトウェアは、各ロジカルプロセッサ上で実行する必要がある。
ただし、全てのロジカルプロセッサである一点で同じ TSC の値を持つようにすることは難しい場合がある。

64-bit の IA32_TSC_ADJUST MSR (アドレス: 3BH) を利用することで、TSC の調整の同期を簡略化することができる。
IA32_TSC_ADJUST MSR は、ロジカルプロセッサごとに管理されており、以下のように動作する。
- RESET 時に IA32_TSC_ADJUST MSR の値は 0 になる。
- IA32_TIME_STAMP_COUNTER MSR への WRMSR 命令の実行により TSC の値を X 増やした (または減らした) 場合、IA32_TSC_ADJUST MSR の値も X 増加 (または減少) する。
- IA32_TSC_ADJUST MSR への WRMSR 命令の実行により MSR の値を X 増やした (または減らした) 場合、IA32_TIME_STAMP_COUNTER MSR の値も X 増加 (または減少) する。

IA32_TIME_STAMP_COUNTER MSR と違い、IA32_TSC_ADJUST MSR の値は WRMSR 命令によってのみ変化するため、複数のロジカルプロセッサで TSC の値を調整したい場合、それらのロジカルプロセッサ上で同じ値を IA32_TSC_ADJUST MSR に設定すれば良い。

CPUID.(EAX=07H,ECX=0H):EBX.TSC_ADJUST[bit 1] のプロセッサで利用可能である。


## Invariant Time-Keeping

Invariant TSC は、Always Running Timer (ART) と呼ばれる Invariant Timekeeping Hardware をベースにしており、コアのクリスタルクロックの周波数で動作する。

CPUID.15H:EBX[31:0] != 0 かつ CPUID.80000007H:EDX.InvariantTSC[bit 8] = 1 の場合、TSC と ART の間には以下の関係式が成り立つ。

TSC_Value = (ART_Value * CPUID.15H:EBX[31:0]) / CPUID.15H:EAX[31:0] + K

ただし、K は特権ソフトウェアが調整できるオフセットである。

ART ハードウェアがリセットされた場合、Invariant TSC および K もまたリセットされる。



# References

- [Intel® 64 and IA-32 Architectures Software Developer’s Manual, Volume 3B: System Programming Guide, Part 2](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3b-part-2-manual.pdf)
