---
title: 【Linux Basics】ファイルタイプ
emoji: "🐧"
type: "idea"
topics: ["linux"]
published: true
---

`ls -l` コマンドを実行した際に、行の一番左に表示される文字が、ファイルタイプを示している。

```
$ ls -l
total 4
brw-r--r-- 1 root   root   101, 101 Aug 14 22:52 block
crw-r--r-- 1 root   root   100, 100 Aug 14 22:52 char
drwxrwxr-x 2 ubuntu ubuntu     4096 Aug 14 22:47 directory
lrwxrwxrwx 1 ubuntu ubuntu        7 Aug 14 22:47 link -> regular
prw-rw-r-- 1 ubuntu ubuntu        0 Aug 14 22:48 pipe
-rw-rw-r-- 1 ubuntu ubuntu        0 Aug 14 22:47 regular
srwxrwxr-x 1 ubuntu ubuntu        0 Aug 14 22:49 socket
```

- `-`: レギュラーファイル
- `d`: ディレクトリ
- `l`: リンクファイル
- `b`: ブロックデバイス
- `c`: キャラクタデバイス
- `p`: 名前付きパイプ
- `s`: ソケットファイル
