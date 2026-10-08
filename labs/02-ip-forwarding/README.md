# 02 - IP Forwarding

## Goal

Linuxは初期状態では通常、別NIC間のパケット転送を行わない。

IP Forwardingを有効化し、

```text
WAN
 ↓
Linux
 ↓
LAN
```

の中継を可能にする。

学習項目：

- RouterとHostの違い
- IP Forwarding
- Linux KernelのPacket Forwarding
