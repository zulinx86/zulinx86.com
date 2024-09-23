---
title: 【仮想化】vsock
emoji: "🐧"
type: "tech"
topics: ["virtualization", "vsock", "networking"]
published: true
---


# 概要

vsock は VM socket の略であり、仮想マシン (VM) とハイパーバイザー間の通信のためのアドレスファミリーである。

仮想マシンとホスト上のユーザースペースアプリケーションの両方が VM socket API を使用して、ゲストとホスト間の高速かつ効率的なコミュニケーションをすることができる。

有効なソケットタイプは、`SOCK_STREAM` (TCP) および `SOCK_DRAM` (UDP) である。

接続の確立は、宛先ソケットアドレスを指定して `connect()` を呼び出すことで可能である。

接続の待機は、`bind()` でソケットアドレスを紐付けた後に `listen()` を呼び出すことで可能である。

データの転送は `send()` / `write()` 系のシステムコールででき、データの受信は `recv()` / `read()` 系のシステムコールで可能である。

ソケットアドレスは、32-bit の Context ID (CID) と 32-bit のポート番号のタプル `(cid, port)` で指定される。
CID は送信元または宛先を指定し、ポート番号はマシン内のサービスを区別する役割を持つ。

```c
struct sockaddr_vm {
    sa_family_t     svm_family;     /* Address family: AF_VSOCK */
    unsigned short  svm_reserved1;  /* Always set to 0 */
    unsigned int    svm_port;       /* Port # in host byte order */
    unsigned int    svm_cid;        /* Address in host byte order */
    unsigned char   svm_zero[sizeof(struct sockaddr) -
                             sizeof(sa_family_t) -
                             sizeof(unsigned short) -
                             sizeof(unsigned int) -
                             sizeof(unsigned int)];
};
```

CID には、いくつか予約された値がある。
- `VMADDR_CID_ANY` (-1): 任意の CID にバインドするのに使用
- `VMADDR_CID_HYPERVISOR` (0): ハイパーバイザーの CID (deprecated)
- `VMADDR_CID_RESERVED` (1): 予約された CID
- `VMADDR_CID_HOST` (2): ホストの CID

Kernel config としては、以下が必要である。
- ホスト側: `CONFIG_VHOST_VSOCK`
- ゲスト側: `CONFIG_VIRTIO_VSOCKETS`

QEMU では `-device vhost-vsock-pci,guest-cid=<cid>` をつけることで vsock を有効にできる。



# ハンズオン

例えば CID 123 を割り当てた場合で、ゲストとホスト間でやりとりするには、以下の通り。

ホスト側 (host.py)

```py
#!/usr/bin/env python3

import socket

CID = socket.VMADDR_CID_HOST
PORT = 9999

s = socket.socket(socket.AF_VSOCK, socket.SOCK_STREAM)
s.bind((CID, PORT))
s.listen()
(conn, (guest_cid, guest_port)) = s.accept()

print(f"Connection established by cid={guest_cid} port={guest_port}")

while True:
    buf = conn.recv(64)
    if not buf:
        break

    print(f"Received bytes: {buf}")
```

ゲスト側 (guest.py)

```py
#!/usr/bin/env python3

import socket

CID = socket.VMADDR_CID_HOST
PORT = 9999

s = socket.socket(socket.AF_VSOCK, socket.SOCK_STREAM)
s.connect((CID, PORT))
s.sendall(b"Hello, world!")
s.close()
```

```
# python3 guest.py
```
```
$ python3 host.py
Connection established by cid=123 port=1752217536
Received bytes: b'Hello, world!'
```


# References

- [VSOCK: Introduce VM Sockets · torvalds/linux@d021c34](https://github.com/torvalds/linux/commit/d021c344051af91f42c5ba9fdedc176740cbd238)
- [vsock(7) - Linux manual page](https://man7.org/linux/man-pages/man7/vsock.7.html)
- [Understanding Vsock. General information | by FrancoisD | Medium](https://medium.com/@F.DL/understanding-vsock-684016cf0eb0)
- [vsock notes](https://gist.github.com/nrdmn/7971be650919b112343b1cb2757a3fe6)
