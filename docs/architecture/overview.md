# Overview

## VM構成

開発初期は、**Windows 11上のVMware Workstationに2台のVMを構築する**。

| VM | 役割 | 仮想NIC |
|---|---|---|
| **VM① Debian Router VM** | 自作ルーター本体 | WAN用 ×1、LAN用 ×1 |
| **VM② Client VM** | ルーター経由の通信を試験する端末 | LAN用 ×1 |

ルーター用Debian VMに、最低2つの仮想NICを持たせる。

```text
Windows 11
│
├─ WSL2
│   └─ 普段のLinux開発・補助作業
│
└─ VMware Workstation
    │
    ├─ VM① Debian Router VM
    │   ├─ 仮想NIC 1：WAN
    │   └─ 仮想NIC 2：LAN
    │
    └─ VM② Client VM
        └─ 仮想NIC 1：LAN
```

WSL2の中にVMを作るのではなく、**Windows上でVMware Workstationを直接使用して2台のVMを構築する**（理由は [002-use-vmware.md](../decisions/002-use-vmware.md) を参照）。

```text
                    Internet
                       │
                VMware NAT等
                       │
                  WAN側 NIC
                       │
             ┌─────────▼─────────┐
             │    Debian VM      │
             │                   │
             │  Router Software  │
             │                   │
             │  iproute2         │
             │  nftables         │
             │  Linux Kernel     │
             └─────────┬─────────┘
                       │
                  LAN側 NIC
                       │
             ┌─────────▼─────────┐
             │    Client VM      │
             │                   │
             │ Debian / Ubuntu   │
             └───────────────────┘
```

ネットワーク構成の詳細は [network-topology.md](network-topology.md) を参照。

---

## 2台のVMの役割

本プロジェクトでは、**「学習環境」と「テスト環境」を完全に分離するわけではない**。

Router VMとClient VMの2台をセットで使いながら学習する。

| VM | 役割 |
|---|---|
| **Debian Router VM** | ルーター機能を設定・実装し、内部動作を観察する実験対象 |
| **Client VM** | Router VMを通る通信を発生させ、設定した機能が正しく動くか確認する端末 |

イメージは以下の通り。

```text
Client VM
   │
   │ ping / curl / traceroute / dig
   ▼
Debian Router VM
   │
   │ Routing / NAT / Firewall / DHCP / DNS
   ▼
Internet
```

したがって、

> **Router VM = 実験対象**

> **Client VM = 実験用の通信を発生させる端末**

と考える。

### Router VM

実験対象。以下を設定・実装し、観察する。

- Routing設定
- IP Forwarding
- NAT
- Firewall
- DHCP
- DNS
- Packet Capture
- 自作Router Software

### Client VM

通信発生用。

Router VMの設定が正しく機能しているかを確認する。

学習はこの2台を組み合わせて行う。
