---
title: 【x86-64】SMAP (Supervisor-Mode Access Prevention) / SMEP (Supervisor-Mode Execution Prevention)
emoji: "🐧"
type: "tech"
topics: ["x86"]
published: true
---


SMEP と SMAP は、それぞれ以下の CPUID ビットでサポートを確認できる。

- SMEP: CPUID.(EAX=07H,ECX=0):EBX.SMEP[bit 7]
- SMAP: CPUID.(EAX=07H,ECX=0):EBX.SMAP[bit 20]

SMEP と SMAP は、それぞれ以下の CR4 レジスタのビットを通じて有効化できる。

- SMEP: CR4.SMEP[bit 20]
- SMAP: CR4.SMAP[bit 21]

SMEP と SMAP は、それぞれ以下のような機能である。

- CR4.SMEP = 1 のとき、スーパーバイザーモードで動作してるソフトウェアは、ユーザーモードでアクセスできるリニアアドレスから命令をフェッチすることができない。
- CR4.SMAP = 1 のとき、スーパーバイザーモードで動作してるソフトウェアは、ユーザーモードでアクセスできるリニアアドレスにあるデータにアクセスすることができない。