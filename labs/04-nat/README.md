# 04 - NAT

## Goal

プライベートネットワークから外部ネットワークへ通信するためにNATを設定する。

```text
10.0.0.10
    ↓
Debian Router
    ↓
Source NAT
    ↓
WAN IP
    ↓
Internet
```

利用：

```text
nftables
```
