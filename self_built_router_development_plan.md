# Self-Built Router Development Plan

## 1. プロジェクト概要

本プロジェクトでは、**ルーターの仕組みを理解すること**を目的として、自作ルーターを開発する。

OSそのものは自作せず、Linuxを土台として利用し、その上でルーターに必要な機能を段階的に構築・実装する。

最終的には、VM上で開発・検証したソフトウェアを、小型PCやSBCなどの実機へ移植する。

---

## 2. 目的

### 主目的

- ルーターがどのようにパケットを転送しているのか理解する
- Linuxがどのようにルーターとして動作するのか理解する
- Routing / NAT / Firewall / DHCP / DNS などの役割を理解する
- ルーターの制御ソフトウェアを自分で実装する
- 仮想環境から実機へ移植する経験を得る

### 今回やらないこと

- OSそのものの自作
- Linuxカーネルそのものの再実装
- NICドライバの自作
- 最初から高性能な10Gbpsルーターを目指す
- 最初からハードウェアを購入する

---

# 3. 「自作」の範囲

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

---

# 4. OS方針

## 採用OS

**Debian**

を採用する。

### Debianを選ぶ理由

- シンプルなLinux環境を構築しやすい
- Linux標準のネットワーク機能を直接学びやすい
- ルーター専用OSによる抽象化が少ない
- `iproute2`、`nftables`、`systemd-networkd` などを直接扱える
- ARM64 / x86-64 の両方で利用しやすい
- 将来的にSBCやミニPCへ移植しやすい

---

## 今回採用しなかったOS

| OS | 特徴 | 今回採用しない理由 |
|---|---|---|
| Ubuntu Server | Debian系で扱いやすい | Netplanなどの追加レイヤーを減らしたい |
| OpenWrt | ルーター用途に特化 | ルーター機能が最初から用意されすぎている |
| VyOS | 業務用ルーターに近い | 「Linuxをルーター化する過程」を学びにくい |
| pfSense / OPNsense | Firewall用途に強い | BSD系で、今回のLinux学習目的とは少し異なる |

---


# 5. 開発マシンの方針

## 採用する開発マシン

**Windows PCをメイン開発マシンとして使用する。**

今回の選定理由は、Windows PCがx86-64で、MacBookがARM64だからではない。

本プロジェクトの初期段階では、

- Routing
- NAT
- Firewall
- DHCP / DNS
- Linux Network Stack
- 自作Router Software

などを学習・実装するため、CPUアーキテクチャの違いは大きな選定要因にはならない。

重要なのは、**VM環境と仮想ネットワークを構築しやすいこと**である。

---

## Windowsを採用する主な理由

WindowsではVMware Workstationを利用し、

- 複数のVM
- 複数の仮想NIC
- NAT Network
- Host-only Network
- Custom Network

などを比較的柔軟に構成できる。

今回必要になる構成は、例えば以下である。

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

このようなネットワーク構成を作りやすいため、Windows + VMware Workstationを採用する。

---

## CPUアーキテクチャについて

手持ち環境では、

| マシン | CPUアーキテクチャ |
|---|---|
| Windows PC | x86-64 / amd64 |
| MacBook | ARM64 / arm64 |

となる。

ただし、**現段階ではこれを開発マシン選定の主な理由にはしない。**

Debianはamd64 / arm64の両方で利用でき、自作ソフトウェアも移植可能な設計にする。

```text
Router Software
      │
      ├── linux/amd64
      │       ↓
      │   x86-64環境
      │
      └── linux/arm64
              ↓
           ARM SBC
```

したがって、

> 開発マシン選定ではCPUアーキテクチャよりも、VM・仮想NIC・仮想ネットワークの作りやすさを優先する。

という方針とする。

---

## MacBookの位置付け

MacBookはメイン開発環境にはしないが、将来的にARM64 SBCへ移植する場合の検証環境として利用できる。

例えば、

```text
Windows
  ↓
Debian amd64 VM
  ↓
Router Software開発
  ↓
ARM64 Build
  ↓
MacBook上のDebian arm64 VM
  ↓
ARM SBC
```

のように、ARM64移植確認用として活用できる。

---

# 6. 開発環境の方針

## 先にハードウェアを買わない

当初は、

- Raspberry Pi
- NanoPi
- N100系ミニPC
- ルーター向けSBC

などを候補として考えていた。

しかし、最初にハードウェアを選定するのではなく、

> **まずDebian VM上でルーターを開発し、その後、必要な性能を見て実機を選ぶ**

方針とする。

---

# 7. VM構成

## 基本構成：VMは2台

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

WSL2の中にVMを作るのではなく、**Windows上でVMware Workstationを直接使用して2台のVMを構築する**。


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

### 学習時の基本的な流れ

例えばRoutingを学ぶ場合は、

1. Router VMで `ip addr` や `ip route` を確認する
2. Router VMでRouting設定を変更する
3. Client VMから `ping` や `traceroute` を実行する
4. Router VMで `tcpdump` などを使ってPacketの流れを観察する

という流れで学習する。

Firewallの場合も同様に、

```text
Router VM
   ↓
Firewall Ruleを設定
   ↓
Client VM
   ↓
curl / pingなどで通信
   ↓
Router VM
   ↓
許可・拒否されたことを確認
```

となる。

したがって、

> **Router VM = 実験対象**

> **Client VM = 実験用の通信を発生させる端末**

と考える。

学習はこの2台を組み合わせて行う。


## 仮想NICの役割

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

# 8. 最初のネットワーク例

## Router VM

```text
WAN : DHCPなどで取得
LAN : 10.0.0.1/24
```

## Client VM

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

# 9. 実装・学習する機能

## Step 1: Network Interface

まずNICとIPアドレスを理解する。

主に使用するもの：

```bash
ip addr
ip link
ip route
```

学習項目：

- NIC
- MAC Address
- IP Address
- Subnet
- Default Gateway

---

## Step 2: IP Forwarding

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

---

## Step 3: Routing

Linux Routing Tableを利用して、

```text
Destination
    ↓
Routing Table
    ↓
Next Hop / Interface
```

という経路選択を理解する。

利用：

```bash
ip route
```

---

## Step 4: NAT

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

---

## Step 5: Firewall

通信の許可・拒否を制御する。

例：

```text
LAN → Internet : Allow
Internet → LAN : Deny
SSH → Router   : 条件付きAllow
```

利用：

```text
nftables
```

---

## Step 6: DHCP

Clientへ自動的に、

- IP Address
- Subnet Mask
- Default Gateway
- DNS Server

を配布する。

最初は既存ソフトを使い、仕組みを理解した後に自作を検討する。

---

## Step 7: DNS

名前解決の役割を理解する。

```text
example.com
    ↓
DNS
    ↓
IP Address
```

ルーター自身をDNS Forwarderとして動作させることも検討する。

---

## Step 8: IPv6

IPv4の仕組みを理解した後、

- IPv6 Addressing
- NDP
- Router Advertisement
- IPv6 Routing
- Firewall

を学習する。

---

## Step 9: VLAN

1台の物理NIC上で複数の論理ネットワークを扱う。

```text
NIC
 │
 ├─ VLAN 10 → Home
 ├─ VLAN 20 → Server
 └─ VLAN 30 → Lab
```

---

## Step 10: VPN

WireGuardなどを使用して、

```text
Remote PC
    ↓
Internet
    ↓
VPN
    ↓
Router
    ↓
LAN
```

を構築する。

---

# 10. 自作Router Software

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

---

# 11. VMを先に使うメリット

## 1. ハードウェア購入が不要

最初から、

- Raspberry Pi
- NanoPi
- Mini PC

などを購入する必要がない。

---

## 2. ネットワークを壊しても戻せる

VM Snapshotを利用できるため、

- Routingを壊した
- Firewallですべて遮断した
- SSHできなくなった
- Network設定を間違えた

場合でも復旧しやすい。

---

## 3. 複数ネットワークを簡単に作れる

VMなら、

```text
WAN
LAN
DMZ
Server Network
Test Network
```

などを容易に追加できる。

---

## 4. 必要スペックを実測できる

開発後に、

```bash
top
htop
free -h
```

などを使用して負荷を測定する。

例えば、

```text
CPU Usage : 5%
RAM Usage : 300MB
```

なら、高性能な実機は不要と判断できる。

逆に、

```text
CPU Usage : 80%
RAM Usage : 1.5GB
```

なら、それに合わせて実機を選定する。

---

# 12. ハードウェア選定は後で行う

開発・負荷測定後に実機を選定する。

候補：

- NanoPi系
- Raspberry Pi系
- ARM SBC
- N100系Mini PC
- Router Appliance

---

## 実機選定時の主な評価項目

| 項目 | 内容 |
|---|---|
| CPU | VMでのCPU使用率を基準に決定 |
| RAM | 実測値＋余裕を持たせる |
| NIC | 最低2ポート |
| NIC速度 | 1GbE / 2.5GbE |
| Architecture | ARM64 / x86-64 |
| Debian対応 | 必須 |
| Storage | microSD / eMMC / SSD |
| 消費電力 | 常時稼働を考慮 |
| サイズ | 小型を優先 |
| 冷却 | ファンレスが理想 |

---

# 13. VMから実機への移植

VMと実機で基本構造は変わらない。

## VM

```text
ens33 → WAN
ens34 → LAN
```

## 実機

```text
eth0 → WAN
eth1 → LAN
```

主な違いは、

- Interface名
- NIC Driver
- MAC Address
- CPU Architecture
- Hardware性能

程度である。

ソフトウェアをハードウェア依存にしなければ、

```text
Debian VM
   ↓
自作Router Software
   ↓
実機Debian
```

へ比較的容易に移植できる。

---

# 14. 開発ロードマップ

```text
Phase 1
Debian VM作成
        │
        ▼
Phase 2
仮想NICを2つ設定
        │
        ▼
Phase 3
Linuxを手動でRouter化
        │
        ▼
Phase 4
Routing / NAT / Firewall理解
        │
        ▼
Phase 5
DHCP / DNS追加
        │
        ▼
Phase 6
Router Software開発
        │
        ▼
Phase 7
VM上で自動テスト
        │
        ▼
Phase 8
CPU / RAM / Network負荷測定
        │
        ▼
Phase 9
必要Hardware Spec決定
        │
        ▼
Phase 10
SBC / Mini PC選定
        │
        ▼
Phase 11
Debianを実機へInstall
        │
        ▼
Phase 12
Router Software移植
        │
        ▼
完成
```

---

# 15. 現時点で決定したこと

| 項目 | 決定内容 |
|---|---|
| プロジェクト目的 | ルーターの仕組みを理解する |
| OS自作 | しない |
| OS | **Debian** |
| ルーター専用OS | 使用しない |
| 開発マシン | **Windows PC** |
| 開発マシン選定理由 | VMware WorkstationでVM・仮想NIC・仮想ネットワークを構築しやすいため |
| CPUアーキテクチャ | 現段階では主要な選定理由にしない |
| MacBookの位置付け | 将来のARM64移植・検証用として利用可能 |
| 開発開始環境 | **Windows 11 + VMware Workstation** |
| 初期VM台数 | **2台（Debian Router VM + Client VM）** |
| Router VM NIC | **2つ（WAN用 + LAN用）** |
| Client VM NIC | **1つ（LAN用）** |
| Router VMの役割 | ルーター機能を設定・実装し、内部を観察する実験対象 |
| Client VMの役割 | 通信を発生させ、Router VMの動作を確認する端末 |
| 学習方法 | Router VMとClient VMをセットで使用する |
| WSL2の位置付け | VM構築には使用せず、普段のLinux開発・補助作業に使用 |
| 最初の開発 | Linuxを手動でルーター化 |
| 自作Software | Linux機能を理解した後に開発 |
| Hardware購入 | **現時点では行わない** |
| Hardware選定 | VMでの開発・負荷測定後 |
| 実機 | 小型SBC / Mini PCを候補とする |
| 最終目標 | VMで作ったRouter Softwareを実機へ移植 |

---

# 16. 次に行うこと

次の作業は、

> **Debian Router VMの構築**

とする。

具体的には、

1. Debian VMを作成
2. 仮想NICを2つ設定
3. WAN / LANネットワークを分離
4. Client VMを作成
5. IP Addressを設定
6. Packet Forwardingを確認
7. Routingを構築
8. NATを設定
9. Internet通信を確認

という順序で進める。

---

## 最終イメージ

```text
                         Internet
                            │
                            │
                     ┌──────▼──────┐
                     │     WAN     │
                     └──────┬──────┘
                            │
                  ┌─────────▼─────────┐
                  │                   │
                  │   Self-Built      │
                  │      Router       │
                  │                   │
                  │      Debian       │
                  │         +         │
                  │      routerd      │
                  │                   │
                  └─────────┬─────────┘
                            │
                           LAN
                            │
              ┌─────────────┼─────────────┐
              │             │             │
             PC          Server          AP
```

まず仮想環境でこの構成を完成させ、その後、必要な性能を見極めて実機へ移植する。
