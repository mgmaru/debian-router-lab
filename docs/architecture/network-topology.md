# Network Topology

ルーターとして構成するネットワークを記録する。

VMwareでの実現方法（VMnetの割り当てなど）は [environment/vmware/network-topology.md](../../environment/vmware/network-topology.md) を参照。

## 基本構成

```text
                    Internet
                       │
                  VMware NAT
                       │
                  WAN側 NIC
                       │
             ┌─────────▼─────────┐
             │   Debian Router   │
             │        VM         │
             └─────────┬─────────┘
                       │
                  LAN側 NIC
                       │
               Virtual Network
                       │
             ┌─────────▼─────────┐
             │    Client VM      │
             └───────────────────┘
```

---

## NICの役割

| NIC | 役割 | 例 |
|---|---|---|
| NIC 1 | WAN | VMware NAT Network |
| NIC 2 | LAN | Host-only / 独立仮想ネットワーク |

ルーターVMが、

```text
Client VM
    ↓
Debian Router VM
    ↓
VMware NAT
    ↓
Internet
```

という経路の中継点になる。

---

## 最初のネットワーク例

### Router VM

```text
WAN : DHCPなどで取得
LAN : 10.0.0.1/24
```

### Client VM

```text
IP      : 10.0.0.10/24
Gateway : 10.0.0.1
```

初期状態では、

```text
Client
  ↓
Router
  ✕
Internet
```

となる。

そこから必要な機能を1つずつ追加していく。

---

## 将来の拡張

将来的には、

```text
                       WAN
                        │
                  Debian Router
               ┌────────┼────────┐
               │        │        │
              LAN      DMZ      Lab
               │        │        │
           Client VM Server VM Test VM
```

のように複数ネットワークへ拡張する可能性もある。
