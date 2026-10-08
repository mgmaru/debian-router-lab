# routerd

Linuxの標準機能を理解した後、それらを制御する自作ソフトウェアを開発する。

仮称：

```text
routerd
```

---

## routerdのイメージ

```text
┌──────────────────────────────┐
│           routerd            │
│                              │
│ Interface Management         │
│ Routing Management           │
│ Firewall Management          │
│ NAT Management               │
│ Traffic Monitoring           │
│ REST API / CLI               │
└──────────────┬───────────────┘
               │
               │ Netlink / nft
               ▼
┌──────────────────────────────┐
│        Linux Kernel          │
│                              │
│ Routing Table                │
│ Netfilter                    │
│ Network Interface            │
└──────────────────────────────┘
```

---

## 開発思想

最初は人間が直接、

```bash
ip route add ...
nft ...
```

と操作する。

その後、

```text
手動操作
   ↓
Linux機能を理解
   ↓
自作プログラムから操作
```

へ移行する。

最終的には、

```text
自作Router Software
        ↓
Netlink / nftables
        ↓
Linux Kernel
        ↓
NIC
```

という構造を目指す。
