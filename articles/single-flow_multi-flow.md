---
title: 【検証】シングルフロートラフィックとマルチフロートラフィックの帯域幅
emoji: "🕸️"
type: "idea"
topics: ["network", "aws", "ec2"]
published: true
---



# Amazon EC2 インスタンスのネットワーク帯域幅

公式ドキュメントに記載の通り、Amazon EC2 インスタンスのネットワーク帯域幅は、シングルフロートラフィック (Single-flow traffic) か、マルチフロートラフィック　(Multi-flow traffic) かで異なります。

- シングルフロートラフィック
    - 最大 5 Gbps まで利用可能
    - クラスタープレイスメントグループを使用した場合、最大 10 Gbps まで利用可能
    - 同じサブネット内かつ ENA Express を設定した場合、最大 25 Gbps まで利用可能
- マルチフロートラフィック: インスタンスで利用可能なネットワーク帯域幅をすべて利用可能

実際には、帯域幅はもっと様々な要素によってキャップされます。詳細については、公式ドキュメントを見てください。
[Amazon EC2 instance network bandwidth - Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-network-bandwidth.html)

この記事では、シングルフロートラフィック or マルチフロートラフィックの上限に関わる挙動のみに焦点を当てて、検証していきます。



# 検証


## 検証方法

ベンチマークのやり方は、以下の AWS re:Post の記事で解説されています。
[Benchmark network throughput between EC2 Linux instances in the same VPC - AWS re:Post](https://repost.aws/knowledge-center/network-throughput-benchmark-linux-ec2)


## 検証環境

2 つの EC2 インスタンスを、以下の条件で起動した検証環境を利用します。

- リージョン: 同一リージョン (us-east-1)
- サブネット: 同一サブネット
- インスタンスタイプ: m6i.metal
- AMI: Amazon Linux 2023

### インスタンスタイプのネットワーク帯域幅

今回、選択した m6i.metal は 50 Gbps までネットワーク帯域幅が利用できます。

```
$ aws ec2 describe-instance-types --filters "Name=instance-type,Values=m6i.*" --query "InstanceTypes[].[InstanceType, NetworkInfo.NetworkPerformance, NetworkInfo.NetworkCards[0].BaselineBandwidthInGbps]" --output table

------------------------------------------------
|             DescribeInstanceTypes            |
+---------------+----------------------+-------+
|  m6i.24xlarge |  37.5 Gigabit        |  None |
|  m6i.large    |  Up to 12.5 Gigabit  |  None |
|  m6i.8xlarge  |  12.5 Gigabit        |  None |
|  m6i.16xlarge |  25 Gigabit          |  None |
|  m6i.32xlarge |  50 Gigabit          |  None |
|  m6i.metal    |  50 Gigabit          |  None |
|  m6i.4xlarge  |  Up to 12.5 Gigabit  |  None |
|  m6i.xlarge   |  Up to 12.5 Gigabit  |  None |
|  m6i.12xlarge |  18.75 Gigabit       |  None |
|  m6i.2xlarge  |  Up to 12.5 Gigabit  |  None |
+---------------+----------------------+-------+
```

### iperf のインストール

インストール方法は、AWS re:Post の記事の通りです。
```
sudo yum groupinstall -y "Development Tools"
sudo yum install -y git

cd /usr/local/
sudo git clone https://git.code.sf.net/p/iperf2/code iperf2-code

cd /usr/local/iperf2-code
sudo ./configure
sudo make
sudo make install
```

以下で、正しくインストールされているか、確認します。
```
$ which iperf
/usr/local/bin/iperf
$ iperf -v
iperf version 2.1.n (6 March 2024) pthreads
```

### iperf サーバーの起動

起動した 2 つの EC2 インスタンスのいずれか一方で、iperf をサーバーモードで起動しておきます。

- `-s`, `--server`: サーバーモードで起動
- `-p`, `--port`: リッスンするポート番号

```
sudo iperf -s -p 5001
```

### iperf クライアントのオプション

iperf クライアント側で使うオプションをここで紹介しておきます。

- `-c`, `--client`: クライアントモードで起動
- `-t`, `--time`: 実行する時間

```
iperf -c <server-private-ip> -t 10
```


## 検証結果

| フロー | クラスタープレイスメントグループ | ENA Express | 最大帯域幅 |
|:---:|:---:|:---:|:---:|
| シングル | なし | なし | 5 Gbps |
| シングル | あり | なし | 10 Gbps |
| シングル | なし | あり | 25 Gbps |
| マルチ | なし | なし | インスタンスの最大帯域幅 (今回は 50 Gbps) |
| マルチ | あり | なし | インスタンスの最大帯域幅 (今回は 50 Gbps) |
| マルチ | なし | あり | インスタンスの最大帯域幅 (今回は 50 Gbps) |


### 検証 1: シングルフロートラフィック

シングルフロートラフィックの最大 5 Gbps の制限に当たっていることがわかります。

```
$ iperf -c <server-private-ip> -t 10
------------------------------------------------------------
Client connecting to <server-private-ip>, TCP port 5001
TCP window size:  325 KByte (default)
------------------------------------------------------------
[  1] local <client-private-ip> port 45002 connected with <server-private-ip> port 5001
[ ID] Interval       Transfer     Bandwidth
[  1] 0.00-10.02 sec  5.78 GBytes  4.96 Gbits/sec
```


### 検証 2: シングルフロートラフィック (クラスタープレイスメントグループ)

2 つの EC2 インスタンスを同じクラスタープレイスメントグループに配置されることで、シングルフロートラフィックであっても、最大 10 Gbps 近くまで発揮できています。

```
$ iperf -c <server-private-ip> -t 10
------------------------------------------------------------
Client connecting to <server-private-ip>, TCP port 5001
TCP window size:  325 KByte (default)
------------------------------------------------------------
[  1] local <client-private-ip> port 56594 connected with <server-private-ip> port 5001
[ ID] Interval       Transfer     Bandwidth
[  1] 0.00-10.01 sec  11.1 GBytes  9.53 Gbits/sec
```


### 検証 3: シングルフロートラフィック (ENA Express 有効)

2 つの EC2 インスタンス上で ENA Express を有効にすることで、シングルフロートラフィックであっても、最大 25 Gbps 近くまで発揮できています。
なお、クラスタープレイスメントグループに配置されている必要はありません。

```
$ iperf -c <server-private-ip> -t 10
------------------------------------------------------------
Client connecting to <server-private-ip>, TCP port 5001
TCP window size:  325 KByte (default)
------------------------------------------------------------
[  1] local <client-private-ip> port 34724 connected with <server-private-ip> port 5001
[ ID] Interval       Transfer     Bandwidth
[  1] 0.00-10.00 sec  27.9 GBytes  23.9 Gbits/sec
```


### 検証 4: マルチフロートラフィック

シングルフロートラフィックで最大 5 Gbps なので、理論的には 10 個の並列したフローがあれば、インスタンスの最大帯域幅 (50 Gbps) まで利用できるはずです。
しっかり 50 Gbps まで利用できています。

```
$ iperf -c <server-private-ip> -t 10 --parallel 10
------------------------------------------------------------
Client connecting to <server-private-ip>, TCP port 5001
TCP window size:  325 KByte (default)
------------------------------------------------------------
[ 10] local <client-private-ip> port 36544 connected with <server-private-ip> port 5001
[  3] local <client-private-ip> port 36464 connected with <server-private-ip> port 5001
[  1] local <client-private-ip> port 36462 connected with <server-private-ip> port 5001
[  8] local <client-private-ip> port 36492 connected with <server-private-ip> port 5001
[  6] local <client-private-ip> port 36534 connected with <server-private-ip> port 5001
[  2] local <client-private-ip> port 36516 connected with <server-private-ip> port 5001
[  5] local <client-private-ip> port 36460 connected with <server-private-ip> port 5001
[  4] local <client-private-ip> port 36518 connected with <server-private-ip> port 5001
[  9] local <client-private-ip> port 36506 connected with <server-private-ip> port 5001
[  7] local <client-private-ip> port 36476 connected with <server-private-ip> port 5001
[ ID] Interval       Transfer     Bandwidth
[  8] 0.00-10.01 sec  5.76 GBytes  4.94 Gbits/sec
[  4] 0.00-10.01 sec  5.76 GBytes  4.94 Gbits/sec
[  5] 0.00-10.01 sec  5.77 GBytes  4.95 Gbits/sec
[  7] 0.00-10.01 sec  5.77 GBytes  4.95 Gbits/sec
[  3] 0.00-10.01 sec  5.78 GBytes  4.96 Gbits/sec
[ 10] 0.00-10.01 sec  5.78 GBytes  4.96 Gbits/sec
[  2] 0.00-10.01 sec  5.74 GBytes  4.93 Gbits/sec
[  9] 0.00-10.01 sec  5.78 GBytes  4.96 Gbits/sec
[  6] 0.00-10.01 sec  5.78 GBytes  4.96 Gbits/sec
[  1] 0.00-10.01 sec  5.78 GBytes  4.96 Gbits/sec
[SUM] 0.00-10.01 sec  57.7 GBytes  49.5 Gbits/sec
```

10 個よりも多いフローにした場合は、インスタンスの最大帯域幅 (50 Gbps) で制限されるはずです。
確かに 30 個のフローを使ったとしても、50 Gbps で制限されています。

```
$ iperf -c <server-private-ip> -t 10 --parallel 30
------------------------------------------------------------
Client connecting to <server-private-ip>, TCP port 5001
TCP window size:  325 KByte (default)
------------------------------------------------------------
[ 27] local <client-private-ip> port 60432 connected with <server-private-ip> port 5001
[  4] local <client-private-ip> port 60136 connected with <server-private-ip> port 5001
[ 13] local <client-private-ip> port 60154 connected with <server-private-ip> port 5001
[ 10] local <client-private-ip> port 60148 connected with <server-private-ip> port 5001
[ 12] local <client-private-ip> port 60138 connected with <server-private-ip> port 5001
[ 14] local <client-private-ip> port 60140 connected with <server-private-ip> port 5001
[  7] local <client-private-ip> port 60144 connected with <server-private-ip> port 5001
[ 16] local <client-private-ip> port 60258 connected with <server-private-ip> port 5001
[ 25] local <client-private-ip> port 60404 connected with <server-private-ip> port 5001
[ 23] local <client-private-ip> port 60420 connected with <server-private-ip> port 5001
[ 20] local <client-private-ip> port 60206 connected with <server-private-ip> port 5001
[  2] local <client-private-ip> port 60146 connected with <server-private-ip> port 5001
[ 19] local <client-private-ip> port 60170 connected with <server-private-ip> port 5001
[  5] local <client-private-ip> port 60142 connected with <server-private-ip> port 5001
[ 24] local <client-private-ip> port 60470 connected with <server-private-ip> port 5001
[ 28] local <client-private-ip> port 60448 connected with <server-private-ip> port 5001
[  1] local <client-private-ip> port 60254 connected with <server-private-ip> port 5001
[  8] local <client-private-ip> port 60260 connected with <server-private-ip> port 5001
[  6] local <client-private-ip> port 60152 connected with <server-private-ip> port 5001
[ 21] local <client-private-ip> port 60192 connected with <server-private-ip> port 5001
[ 30] local <client-private-ip> port 60482 connected with <server-private-ip> port 5001
[ 29] local <client-private-ip> port 60462 connected with <server-private-ip> port 5001
[  3] local <client-private-ip> port 60190 connected with <server-private-ip> port 5001
[ 11] local <client-private-ip> port 60356 connected with <server-private-ip> port 5001
[ 17] local <client-private-ip> port 60478 connected with <server-private-ip> port 5001
[ 26] local <client-private-ip> port 60406 connected with <server-private-ip> port 5001
[ 15] local <client-private-ip> port 60150 connected with <server-private-ip> port 5001
[ 18] local <client-private-ip> port 60176 connected with <server-private-ip> port 5001
[  9] local <client-private-ip> port 60256 connected with <server-private-ip> port 5001
[ 22] local <client-private-ip> port 60204 connected with <server-private-ip> port 5001
[ ID] Interval       Transfer     Bandwidth
[ 24] 0.00-10.01 sec  1.46 GBytes  1.25 Gbits/sec
[ 14] 0.00-10.01 sec  2.55 GBytes  2.19 Gbits/sec
[ 21] 0.00-10.01 sec  1.58 GBytes  1.35 Gbits/sec
[ 26] 0.00-10.01 sec  1.09 GBytes   932 Mbits/sec
[ 10] 0.00-10.01 sec  2.50 GBytes  2.14 Gbits/sec
[  6] 0.00-10.01 sec  1.38 GBytes  1.19 Gbits/sec
[ 12] 0.00-10.01 sec  1.29 GBytes  1.10 Gbits/sec
[ 11] 0.00-10.01 sec  1.20 GBytes  1.03 Gbits/sec
[ 23] 0.00-10.01 sec  3.98 GBytes  3.42 Gbits/sec
[ 17] 0.00-10.01 sec  1.31 GBytes  1.13 Gbits/sec
[ 16] 0.00-10.01 sec  2.47 GBytes  2.12 Gbits/sec
[ 25] 0.00-10.01 sec  1.21 GBytes  1.04 Gbits/sec
[  5] 0.00-10.01 sec  2.54 GBytes  2.18 Gbits/sec
[ 22] 0.00-10.01 sec  5.78 GBytes  4.96 Gbits/sec
[ 28] 0.00-10.01 sec  1.97 GBytes  1.69 Gbits/sec
[  4] 0.00-10.01 sec  1.31 GBytes  1.13 Gbits/sec
[  2] 0.00-10.01 sec  1.09 GBytes   932 Mbits/sec
[  7] 0.00-10.01 sec  1.18 GBytes  1.02 Gbits/sec
[  3] 0.00-10.01 sec  1.70 GBytes  1.46 Gbits/sec
[ 29] 0.00-10.01 sec  1.24 GBytes  1.06 Gbits/sec
[ 15] 0.00-10.01 sec  2.50 GBytes  2.14 Gbits/sec
[ 27] 0.00-10.01 sec  1.87 GBytes  1.61 Gbits/sec
[ 20] 0.00-10.01 sec  1.91 GBytes  1.64 Gbits/sec
[ 18] 0.00-10.01 sec  1.52 GBytes  1.30 Gbits/sec
[ 19] 0.00-10.01 sec  3.64 GBytes  3.13 Gbits/sec
[ 30] 0.00-10.03 sec  1.30 GBytes  1.12 Gbits/sec
[  8] 0.00-10.03 sec  1.44 GBytes  1.24 Gbits/sec
[  9] 0.00-10.01 sec  2.36 GBytes  2.03 Gbits/sec
[ 13] 0.00-10.01 sec  1.19 GBytes  1.02 Gbits/sec
[  1] 0.00-10.03 sec  1.28 GBytes  1.10 Gbits/sec
[SUM] 0.00-10.03 sec  57.8 GBytes  49.6 Gbits/sec
```


### 検証 5: マルチフロートラフィック (クラスタープレイスメントグループ)

クラスタープレイスメントグループに配置されていたとしても、インスタンスの最大帯域幅 (50 Gbps) で制限されます。

```
$ iperf -c <server-private-ip> -t 10 --parallel 10
------------------------------------------------------------
Client connecting to <server-private-ip>, TCP port 5001
TCP window size:  325 KByte (default)
------------------------------------------------------------
[  3] local <client-private-ip> port 44864 connected with <server-private-ip> port 5001
[  2] local <client-private-ip> port 44852 connected with <server-private-ip> port 5001
[  1] local <client-private-ip> port 44838 connected with <server-private-ip> port 5001
[  7] local <client-private-ip> port 44892 connected with <server-private-ip> port 5001
[  6] local <client-private-ip> port 44884 connected with <server-private-ip> port 5001
[  4] local <client-private-ip> port 44880 connected with <server-private-ip> port 5001
[  9] local <client-private-ip> port 44896 connected with <server-private-ip> port 5001
[  8] local <client-private-ip> port 44912 connected with <server-private-ip> port 5001
[  5] local <client-private-ip> port 44866 connected with <server-private-ip> port 5001
[ 10] local <client-private-ip> port 44922 connected with <server-private-ip> port 5001
[ ID] Interval       Transfer     Bandwidth
[  7] 0.00-10.02 sec  5.79 GBytes  4.97 Gbits/sec
[  4] 0.00-10.02 sec  3.05 GBytes  2.62 Gbits/sec
[  2] 0.00-10.02 sec  11.1 GBytes  9.52 Gbits/sec
[  8] 0.00-10.02 sec  3.17 GBytes  2.72 Gbits/sec
[  9] 0.00-10.02 sec  3.24 GBytes  2.78 Gbits/sec
[  3] 0.00-10.02 sec  3.01 GBytes  2.58 Gbits/sec
[ 10] 0.00-10.02 sec  5.55 GBytes  4.76 Gbits/sec
[  1] 0.00-10.02 sec  11.1 GBytes  9.52 Gbits/sec
[  6] 0.00-10.02 sec  5.77 GBytes  4.95 Gbits/sec
[  5] 0.00-10.02 sec  6.02 GBytes  5.16 Gbits/sec
[SUM] 0.00-10.02 sec  57.8 GBytes  49.6 Gbits/sec
```


### 検証 6: マルチフロートラフィック (ENA Express 有効)

ENA Express を有効にしてマルチフロートラフィックを使った場合も、インスタンスの最大帯域幅 (50 Gbps) で制限されます。

```
$ iperf -c <server-private-ip> -t 10 --parallel 10
------------------------------------------------------------
Client connecting to 172.31.70.174, TCP port 5001
TCP window size:  325 KByte (default)
------------------------------------------------------------
[  9] local <client-private-ip> port 59386 connected with <server-private-ip> port 5001
[  1] local <client-private-ip> port 59346 connected with <server-private-ip> port 5001
[  2] local <client-private-ip> port 59388 connected with <server-private-ip> port 5001
[  3] local <client-private-ip> port 59344 connected with <server-private-ip> port 5001
[  8] local <client-private-ip> port 59362 connected with <server-private-ip> port 5001
[  6] local <client-private-ip> port 59374 connected with <server-private-ip> port 5001
[ 10] local <client-private-ip> port 59428 connected with <server-private-ip> port 5001
[  5] local <client-private-ip> port 59366 connected with <server-private-ip> port 5001
[  4] local <client-private-ip> port 59398 connected with <server-private-ip> port 5001
[  7] local <client-private-ip> port 59414 connected with <server-private-ip> port 5001
[ ID] Interval       Transfer     Bandwidth
[  5] 0.00-10.01 sec  4.83 GBytes  4.14 Gbits/sec
[  3] 0.00-10.01 sec  9.94 GBytes  8.53 Gbits/sec
[  7] 0.00-10.01 sec  4.58 GBytes  3.93 Gbits/sec
[  1] 0.00-10.01 sec  2.97 GBytes  2.55 Gbits/sec
[ 10] 0.00-10.01 sec  10.5 GBytes  8.99 Gbits/sec
[  9] 0.00-10.01 sec  3.33 GBytes  2.85 Gbits/sec
[  8] 0.00-10.01 sec  3.39 GBytes  2.91 Gbits/sec
[  2] 0.00-10.01 sec  8.97 GBytes  7.69 Gbits/sec
[  6] 0.00-10.01 sec  4.48 GBytes  3.85 Gbits/sec
[  4] 0.00-10.01 sec  4.72 GBytes  4.05 Gbits/sec
[SUM] 0.00-10.01 sec  57.7 GBytes  49.5 Gbits/sec
```
