# debian-router-lab

Debian上でルーター機能を構築・観察・実装しながら、Linuxルーターの仕組みを理解するための学習用リポジトリです。

このプロジェクトでは、OSやLinuxカーネルそのものは自作せず、Debianを土台として利用します。  
その上で、Routing / NAT / Firewall / DHCP / DNS などのルーター機能を段階的に学習し、最終的にはそれらを制御する自作Router Softwareの開発を目指します。

---

## Goals

このリポジトリの主な目的は以下です。

- Linuxがどのようにルーターとして動作するのか理解する
- NIC、IP Address、Routing Table、Packet Forwardingの関係を理解する
- NAT / Firewall / DHCP / DNS の役割を理解する
- `iproute2`、`nftables`、`tcpdump` などを実際に操作する
- Packetの流れを観察しながらネットワークを理解する
- 手動設定から自動化、自作Router Softwareへ段階的に発展させる
- VM上で開発した構成を将来的にSBC / Mini PCへ移植する

---

## Non-Goals

現時点では以下は対象外とします。

- OSそのものの自作
- Linux Kernelの再実装
- NIC Driverの自作
- 最初から高性能な商用ルーター相当を目指すこと
- 最初から10Gbpsなどの高スループットを追求すること

---

# Development Environment

開発環境は以下を基本構成とします。

```text
Windows 11
│
├─ WSL2
│   └─ 普段のLinux開発・補助作業
│
└─ VMware Workstation
    │
    ├─ VM 1: Debian Router
    │   ├─ Virtual NIC 1: WAN
    │   └─ Virtual NIC 2: LAN
    │
    └─ VM 2: Client
        └─ Virtual NIC 1: LAN
```

VMはWSL2内ではなく、Windows上のVMware Workstationに直接構築します。

---

## VM Roles

2台のVMはセットで使用します。

| VM | Role |
|---|---|
| Debian Router VM | ルーター機能を設定・実装し、内部動作を観察する実験対象 |
| Client VM | Router VMを通る通信を発生させ、動作を確認する端末 |

通信イメージ：

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

Router VMで設定を変更し、Client VMから通信を発生させ、Router VM側でPacketを観察する、という流れを基本とします。

---

# Learning Approach

このプロジェクトでは、単にコマンドを覚えるのではなく、

```text
Concept
   ↓
Configuration
   ↓
Traffic Generation
   ↓
Observation
   ↓
Understanding
```

という流れで学習します。

例えばRoutingを学習する場合：

1. Routingの仕組みを理解する
2. Router VMで `ip route` を確認する
3. Routing Tableを変更する
4. Client VMから `ping` / `traceroute` を実行する
5. Router VMで `tcpdump` を使ってPacketを観察する
6. 結果と理解したことをLabに記録する

---

# Repository Structure

```text
debian-router-lab/
│
├── README.md
│
├── docs/
│   ├── architecture/
│   │   ├── overview.md
│   │   ├── network-topology.md
│   │   └── router-boundary.md
│   │
│   ├── concepts/
│   │   ├── interface.md
│   │   ├── ip-forwarding.md
│   │   ├── routing.md
│   │   ├── nat.md
│   │   ├── firewall.md
│   │   ├── dhcp.md
│   │   └── dns.md
│   │
│   └── decisions/
│       ├── 001-use-debian.md
│       ├── 002-use-vmware.md
│       └── 003-hardware-later.md
│
├── labs/
│   ├── 01-network-interface/
│   │   └── README.md
│   ├── 02-ip-forwarding/
│   │   └── README.md
│   ├── 03-routing/
│   │   └── README.md
│   ├── 04-nat/
│   │   └── README.md
│   ├── 05-firewall/
│   │   └── README.md
│   ├── 06-dhcp/
│   │   └── README.md
│   ├── 07-dns/
│   │   └── README.md
│   ├── 08-ipv6/
│   │   └── README.md
│   ├── 09-vlan/
│   │   └── README.md
│   └── 10-vpn/
│       └── README.md
│
├── environment/
│   ├── vmware/
│   │   ├── README.md
│   │   └── network-topology.md
│   │
│   └── debian/
│       ├── README.md
│       └── packages.txt
│
├── configs/
│   ├── sysctl/
│   ├── network/
│   ├── nftables/
│   ├── dhcp/
│   └── dns/
│
├── scripts/
│   ├── setup/
│   ├── test/
│   └── cleanup/
│
├── src/
│   └── routerd/
│       └── README.md
│
├── deploy/
│   ├── README.md
│   ├── install.sh
│   ├── uninstall.sh
│   └── systemd/
│       └── routerd.service
│
├── tests/
│   ├── integration/
│   └── scenarios/
│
├── captures/
│   └── README.md
│
├── .gitignore
└── LICENSE
```

---

# Directory Responsibilities

## `docs/`

知識・設計・意思決定を記録します。

### `docs/architecture/`

プロジェクト全体の構成や責務を記録します。

例：

- Router VM / Client VMの構成
- WAN / LANの構成
- Linuxに任せる部分と自作する部分

### `docs/concepts/`

各ネットワーク技術の概念を整理します。

```text
docs/concepts/routing.md
```

では「Routingとは何か」を整理し、

```text
labs/03-routing/
```

では「Routingを実際に試した結果」を記録します。

### `docs/decisions/`

重要な技術選定や方針を記録します。

例：

- Debianを採用した理由
- VMware Workstationを利用する理由
- ハードウェア選定を後回しにする理由

---

## `labs/`

このリポジトリの中心となるディレクトリです。

各Labでは、

```text
仮説
 ↓
操作
 ↓
通信
 ↓
観察
 ↓
結果
 ↓
理解
```

を記録します。

各LabのREADMEは以下のような構成を基本とします。

```markdown
# Lab Title

## Goal

このLabで理解すること。

## Topology

使用するネットワーク構成。

## Commands

使用したコマンド。

## Experiment

実際に行った操作。

## Observation

tcpdumpなどで観察した内容。

## Result

実験結果。

## Learned

理解したこと。
```

---

## `environment/`

開発環境を再構築するための情報を保存します。

VMそのものの巨大なファイルはGitに保存せず、

- VM構成
- Virtual NIC構成
- VMware Network設定
- Debian初期設定
- 必要Package

などを記録します。

---

## `configs/`

学習後に再利用可能になった設定ファイルを保存します。

例：

```text
configs/sysctl/99-router.conf
configs/nftables/nftables.conf
configs/network/
```

役割の違い：

```text
labs/
→ なぜその設定が必要なのか

configs/
→ 実際に利用する設定
```

---

## `scripts/`

繰り返し行う操作を自動化します。

例：

- Router初期設定
- Network状態確認
- Test Traffic生成
- 設定のReset

最初からすべて自動化するのではなく、まず手動で仕組みを理解した後にScript化します。

---

## `src/routerd/`

将来的に開発する自作Router Softwareを配置します。

初期段階ではRouter Softwareを作ることを急がず、

```text
Manual Operation
     ↓
Shell Script
     ↓
Router Controller
```

と段階的に進めます。

将来的なイメージ：

```text
routerd
│
├─ Interface Management
├─ Routing Management
├─ Firewall Management
├─ NAT Management
├─ Traffic Monitoring
└─ CLI / API
        │
        ▼
Netlink / nftables
        │
        ▼
Linux Kernel
```

---

## `deploy/`

VM上で開発・検証したルーター機能を、最終的に物理マシンへ導入するためのファイルを管理します。

このプロジェクトでは、VM上で動作した時点を完成とはせず、

> **物理マシン上のDebianへ移植し、実際のWAN / LAN NICを使ってルーターとして動作させること**

を最終ゴールとします。

役割の整理：

```text
src/
→ 実行するRouter Software本体

configs/
→ nftables / sysctl / networkなどの設定

scripts/
→ 開発・検証・補助作業の自動化

deploy/
→ 実機へどのように導入・配置・起動するか
```

実機への移植イメージ：

```text
debian-router-lab
│
├─ src/routerd/
│      │
│      └── build
│            ↓
│      /usr/local/bin/routerd
│
├─ configs/
│      │
│      ├── nftables
│      ├── sysctl
│      └── network
│            ↓
│          /etc/...
│
├─ scripts/
│      └── 必要な補助処理
│
└─ deploy/
       │
       ├── install.sh
       ├── uninstall.sh
       └── systemd/routerd.service
              ↓
        Physical Debian Router
```

想定する実機側の配置例：

```text
/usr/local/bin/
└── routerd

/etc/routerd/
└── routerd.conf

/etc/nftables.conf

/etc/sysctl.d/
└── 99-router.conf

/etc/systemd/system/
└── routerd.service
```

`deploy/` は、VM環境固有の設定を保存する場所ではなく、**実機へ再現可能な形で導入するための仕組み**を管理する場所とします。

---

## `tests/`

Router Softwareや設定の自動テストを保存します。

### `tests/integration/`

複数機能を組み合わせたテスト。

### `tests/scenarios/`

具体的な通信シナリオを使ったテスト。

例：

```text
Client
 ↓
Router
 ↓
Internet
```

において、

- ICMPが通る
- DNSが解決できる
- HTTP/HTTPS通信ができる
- 特定通信がFirewallで拒否される

などを確認します。

---

## `captures/`

Packet Captureに関するメモやサンプルを管理します。

`.pcap` ファイルは容量や機密情報に注意し、必要に応じて `.gitignore` の対象とします。

---

# Learning Roadmap

```text
01 Network Interface
        ↓
02 IP Forwarding
        ↓
03 Routing
        ↓
04 NAT
        ↓
05 Firewall
        ↓
06 DHCP
        ↓
07 DNS
        ↓
08 IPv6
        ↓
09 VLAN
        ↓
10 VPN
        ↓
Automation
        ↓
routerd
        ↓
Physical Router
```

---

# Initial Labs

## 01 - Network Interface

学習内容：

- NIC
- MAC Address
- IPv4 Address
- Subnet
- Default Gateway
- `ip addr`
- `ip link`

---

## 02 - IP Forwarding

学習内容：

- HostとRouterの違い
- Linux Packet Forwarding
- `net.ipv4.ip_forward`

---

## 03 - Routing

学習内容：

- Routing Table
- Destination Network
- Next Hop
- Default Route
- Longest Prefix Match
- `ip route`

---

## 04 - NAT

学習内容：

- Private / Public IP
- SNAT
- MASQUERADE
- Port Translation
- Connection Tracking

---

## 05 - Firewall

学習内容：

- Netfilter
- nftables
- Input / Forward / Output
- Allow / Drop
- Stateful Firewall

---

## 06 - DHCP

学習内容：

- DHCP Discover
- Offer
- Request
- ACK
- IP Address配布
- Default Gateway配布
- DNS Server配布

---

## 07 - DNS

学習内容：

- Name Resolution
- Recursive Resolver
- Authoritative DNS
- DNS Forwarder

---

# Development Policy

## Understand Before Automating

最初からScriptやSoftwareに処理を任せません。

まず手動で、

```bash
ip
nft
sysctl
tcpdump
```

などを操作し、何が起きているのか理解します。

その後で自動化します。

---

## Observe Packets

設定結果だけではなく、可能な限りPacketを観察します。

```bash
tcpdump
```

などを使用し、

```text
Client
   ↓
Router
   ↓
WAN
```

のどこをPacketが通っているのか確認します。

---

## Separate Knowledge and Experiments

```text
docs/
→ 理論・概念

labs/
→ 実験・観察

configs/
→ 再利用する設定

src/
→ 自作Software

scripts/
→ 開発・検証の補助

deploy/
→ 物理マシンへの導入
```

という責務を維持します。

---

# Hardware Strategy

初期段階では実機を購入しません。

まずVM上で、

- CPU Usage
- Memory Usage
- Network Throughput
- 必要NIC数
- 必要Storage

を確認します。

その結果をもとに、

- ARM SBC
- NanoPi
- Raspberry Pi
- N100 Mini PC
- Router Appliance

などから最終的なHardwareを選定します。

```text
Requirements
    ↓
VM Development
    ↓
Load Measurement
    ↓
Hardware Requirements
    ↓
Hardware Selection
    ↓
Deploy Design
    ↓
Physical Debian Router
    ↓
Real WAN / LAN Test
```

---

# Future Goal

このプロジェクトの最終目標は、VM上でルーター機能を学習・開発することだけではありません。

**VMで検証した構成・設定・自作Softwareを実際の小型PC / SBCへ移植し、物理NICを使うDebian Routerとして完成させること**をゴールとします。

最終的には、

```text
Internet
   │
   │ WAN
   ▼
┌─────────────────────┐
│   Physical Router   │
│                     │
│ Debian              │
│ +                   │
│ routerd             │
│                     │
│ Routing             │
│ NAT                 │
│ Firewall            │
└─────────┬───────────┘
          │ LAN
          ▼
        Switch
          │
    ┌─────┼─────┐
    PC  Server  AP
```

という構成へ移植することを目指します。

---

# Status

現在は以下のフェーズです。

```text
[ ] Debian Router VM作成
[ ] Client VM作成
[ ] Virtual Network構築
[ ] Network Interface確認
[ ] IP Forwarding
[ ] Routing
[ ] NAT
[ ] Firewall
[ ] DHCP
[ ] DNS
[ ] IPv6
[ ] VLAN
[ ] VPN
[ ] Automation
[ ] routerd
[ ] Hardware選定
[ ] deploy設計
[ ] Physical Routerへの移植
[ ] 実NICでWAN / LAN接続
[ ] 実機でRouting / NAT / Firewall確認
```
