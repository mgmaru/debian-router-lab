# VMware Network Topology

[docs/architecture/network-topology.md](../../docs/architecture/network-topology.md) の構成を、VMware Workstationの仮想ネットワークで実現する。

## VMnetについて

VMware Workstationでは、VM同士を接続するための仮想ネットワークを作成できる。

今回の構成では、

```text
VMnet8 = WAN側
VMnet2 = LAN側
```

として使用する。

### VMnet8

通常、VMwareのNATネットワークとして使用される。

```text
Debian Router
      │
    VMnet8
      │
Windows
      │
Internet
```

Router VMのWAN側NICを接続する。

### VMnet2

LAN用として作成したCustom Network。

```text
Debian Router
      │
   LAN NIC
      │
    VMnet2
      │
Debian Client
```

実機でいう、

```text
Router
  │
Switch
  │
Client
```

の「LANケーブル + Switch」の部分を仮想的に再現する。

---

## 仮想ネットワーク構成

```text
                    Internet
                       │
                    VMnet8
                     NAT
                       │
                    WAN NIC
                       │
              ┌────────▼────────┐
              │ debian-router   │
              │                 │
              │ WAN         LAN │
              └─────────┬───────┘
                        │
                     VMnet2
                        │
              ┌─────────▼───────┐
              │ debian-client   │
              │                 │
              │ NIC ×1          │
              └─────────────────┘
```

重要なのは、ClientからInternetへ直接接続しないこと。

```text
Client
  ↓
Router
  ↓
Internet
```

となる構成を作る。
