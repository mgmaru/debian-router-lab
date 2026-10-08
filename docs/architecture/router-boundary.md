# Router Boundary

自作ルーターでは、どこまで自分で作るかによって「自作度」が変わる。

今回の方針は、**Linux Kernel以下は既存のものを利用し、ルーターとしての制御部分を理解・実装する**ことである。

```text
┌──────────────────────────────┐
│       自作する領域           │
│                              │
│ Router Controller            │
│ Routing Logic                │
│ Firewall Logic               │
│ Monitoring                   │
│ DHCP / DNS（必要に応じて）   │
├──────────────────────────────┤
│       Linux標準機能          │
│                              │
│ iproute2                     │
│ nftables                     │
│ Netlink                      │
│ Linux Routing Table          │
├──────────────────────────────┤
│        Linux Kernel          │
│                              │
│ TCP/IP Stack                 │
│ Netfilter                    │
│ NIC Driver                   │
├──────────────────────────────┤
│          Hardware            │
└──────────────────────────────┘
```
