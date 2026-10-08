# Debian Router Lab - Setup Progress

## 1. 今回の目的

Debian上でルーター機能を学習・実装するための仮想環境を、Windows 11 + VMware Workstation Pro 上に構築した。

最終的には、VM上で学習・開発したルーター機能を物理マシンへ移植する。

---

## 2. 今回決めた開発構成

```text
MacBook
   │
   │ SSH over Tailscale
   ▼
Windows 11
   │
   │ VMware Workstation Pro
   │
   ├───────────────┐
   ▼               ▼
Debian Router VM   Debian Client VM
```

### 各マシンの役割

| マシン | 役割 |
|---|---|
| MacBook | 外出先からの操作端末 |
| Windows 11 | VMware Workstationを動かすホスト |
| Debian Router VM | ルーター機能の学習・実装対象 |
| Debian Client VM | Router VMを通る通信を発生させる試験端末 |

---

## 3. Windowsへのリモート接続

MacBookからWindowsへ、Tailscaleネットワーク経由でSSH接続できることを確認した。

Windows側のユーザー確認：

```cmd
whoami
```

結果：

```text
alienkaji\kajih
```

この場合、

```text
alienkaji = Windowsのコンピューター名
kajih     = Windowsのユーザー名
```

となる。

したがって、MacBookからの接続は次の形式となる。

```bash
ssh kajih@<WindowsのTailscale IP>
```

---

## 4. VMware Workstation Pro

Windows 11へ VMware Workstation Pro 26H1u1 をインストールした。

VMware Workstation Pro上に、以下2台のVMを構築した。

```text
VM 1: debian-router
VM 2: debian-client
```

---

## 5. Debian Router VM

### VM設定

| 項目 | 設定 |
|---|---|
| VM名 | `debian-router` |
| OS | Debian 13 |
| Architecture | amd64 |
| CPU | 2 vCPU |
| RAM | 2 GB |
| Disk | 20 GB |
| Disk形式 | Single file |
| 初期NIC | NAT |

使用したISO：

```text
debian-13.7.0-amd64-netinst.iso
```

### Debianインストール設定

| 項目 | 設定 |
|---|---|
| Language | English |
| Location | Japan |
| Locale | en_US.UTF-8 |
| Keyboard | American English |
| Hostname | `debian-router` |
| Root password | 未設定 |
| User | `kajih` |
| Partition | Guided - use entire disk |
| Layout | All files in one partition |
| Package survey | No |
| SSH server | Install |
| Standard system utilities | Install |
| Desktop environment | Installしない |
| GRUB | Install |

---

## 6. Debian Client VM

### VM設定

| 項目 | 設定 |
|---|---|
| VM名 | `debian-client` |
| OS | Debian 13 |
| Architecture | amd64 |
| CPU | 1 vCPU |
| RAM | 1 GB |
| Disk | 10 GB |
| Disk形式 | Single file |
| NIC | 1枚 |
| インストール時NIC | NAT |

Client VMもGUIなしの最小Debian構成とした。

主な用途：

```text
ping
curl
traceroute
dig
```

などを使ってRouter VMへ通信を発生させる。

---

## 7. Router VM / Client VMの関係

今回の2台は、

> 学習環境とテスト環境を完全に分離する

のではなく、

> Router VMとClient VMをセットで使って学習する

という位置付け。

```text
Client VM
   │
   │ 通信を発生
   ▼
Router VM
   │
   │ Routing / NAT / Firewallなど
   ▼
Internet
```

### Router VM

実験対象。

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

---

## 8. VMnetについて

VMware Workstationでは、VM同士を接続するための仮想ネットワークを作成できる。

今回の構成では、

```text
VMnet8 = WAN側
VMnet2 = LAN側
```

として使用する予定。

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

今回、LAN用として作成するCustom Network。

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

## 9. 最終的な仮想ネットワーク構成

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

---

## 10. SSH運用

VMwareのコンソールは操作しにくいため、普段の操作はSSHを使用する。

外出先からは、

```text
MacBook
   ↓
Tailscale
   ↓
Windows
   ↓
SSH ProxyJump
   ↓
Debian Router VM
```

という構成を利用できる。

Router VMへTailscaleを直接入れず、Windowsを踏み台にすることで、Router VMのネットワーク構成をシンプルに保つ。

---


## 10.1 Router VMへのSSH経路

現在、Windowsから `debian-router` へSSHする際の入口は、Router VMの**WAN側NIC**である。

構成：

```text
Windows 11
   │
   │ SSH
   ▼
VMware Network Adapter VMnet8
   │
   ▼
VMnet8 (NAT)
   │
   ▼
debian-router
   │
   └─ WAN NIC
        └─ sshd : TCP/22
```

Router VMのWAN側NICには、VMnet8からIPアドレスが付与される。

例えば、

```bash
ip -br addr
```

で、

```text
ens33   UP   192.168.x.x/24
```

のように表示される場合、WindowsからはそのIPへSSHする。

```cmd
ssh kajih@192.168.x.x
```

MacBookから外出先で接続する場合は、

```text
MacBook
   │
   │ Tailscale
   ▼
Windows
   │
   │ SSH ProxyJump
   ▼
VMnet8
   │
   ▼
debian-router WAN NIC
```

という経路になる。

したがって現時点では、

```text
管理用SSH通信
→ WAN側（VMnet8）

Router / Client間の学習用通信
→ LAN側（VMnet2）
```

という役割分担になる。

Firewall学習時にWAN側のTCP/22を誤って遮断するとSSHできなくなる可能性があるため、VMware Consoleは復旧手段として残しておく。

---

## 10.2 Client VMへのSSH

Client VMにもSSH Serverをインストールしているため、**ClientへSSHできるようにしておく**。

ClientはRouterのLAN側にのみ接続し、管理用NICを追加しない。

最終構成：

```text
Windows
   │
   │ SSH
   ▼
debian-router
   │
   │ SSH over VMnet2
   ▼
debian-client
```

ClientにLAN側IPとして、

```text
10.0.0.10/24
```

を設定した後、Router VMから、

```bash
ssh kajih@10.0.0.10
```

で接続できるようにする。

これにより、Router VMへSSHした状態からClient側の操作も行える。

例えばClient側で、

```bash
ping 10.0.0.1
traceroute 8.8.8.8
curl https://example.com
dig example.com
```

などを実行し、Router側では同時に、

```bash
tcpdump
ip route
nft list ruleset
```

などを確認できる。

### Clientへ直接管理用NICを追加しない理由

ClientにVMnet8などの管理用NICを追加すると、

```text
Client
 ├─ LAN → Router
 └─ NAT → Internet
```

のようにRouterを迂回する経路ができてしまい、学習用ネットワークが分かりにくくなる。

そのため、

> **ClientはLAN用NIC 1枚のみ**

とし、

> **ClientへのSSHはRouterを踏み台にして行う**

方針とする。

MacBookから最終的にClientへ入る場合は、

```text
MacBook
   ↓
Windows
   ↓
debian-router
   ↓
debian-client
```

という多段SSH構成にできる。

---


## 10.3 MacBookのSSH Config

外出先のMacBookから毎回長いSSHコマンドを入力しなくて済むように、`~/.ssh/config` に接続先を登録する。

MacBookで以下を開く。

```bash
nano ~/.ssh/config
```

またはVS Codeなどのエディタで `~/.ssh/config` を編集する。

### 設定例

以下のIPアドレスは環境に合わせて置き換える。

- `100.80.124.76` : WindowsのTailscale IP
- `192.168.x.x` : Debian Router VMのWAN側IP
- `10.0.0.10` : Debian Client VMのLAN側IP

```sshconfig
Host windows-home
    HostName 100.80.124.76
    User kajih

Host debian-router
    HostName 192.168.x.x
    User kajih
    ProxyJump windows-home

Host debian-client
    HostName 10.0.0.10
    User kajih
    ProxyJump windows-home,debian-router
```

設定ファイルの権限も確認する。

```bash
chmod 600 ~/.ssh/config
```

---

### Windowsへ接続

```bash
ssh windows-home
```

通信経路：

```text
MacBook
   ↓ Tailscale
Windows
```

---

### Router VMへ接続

```bash
ssh debian-router
```

通信経路：

```text
MacBook
   ↓ Tailscale
Windows
   ↓ VMnet8
Debian Router VM
```

`debian-router` の `HostName` には、Router VMのWAN側IPを設定する。

---

### Client VMへ接続

Client VMのLAN側IPを `10.0.0.10` とした場合、

```bash
ssh debian-client
```

で接続できる構成を目指す。

通信経路：

```text
MacBook
   ↓
Windows
   ↓
Debian Router VM
   ↓ VMnet2
Debian Client VM
```

Client VMはVMnet2側のNIC 1枚だけを持ち、WindowsやVMnet8へ直接接続しない。

これにより、Clientへの管理通信もRouterを経由するため、学習用ネットワーク構成を維持できる。

---

### 初期段階の接続確認

まずは段階的に確認する。

```bash
ssh windows-home
```

が成功することを確認した後、

```bash
ssh debian-router
```

を確認する。

ClientのLAN側IP設定完了後に、

```bash
ssh debian-client
```

を確認する。

問題が起きた場合は、一段ずつ接続して原因を切り分ける。

```text
1. Mac → Windows
2. Windows → Router
3. Router → Client
```

---


### SSH接続確認済み

以下の接続は実際に確認済み。

```bash
ssh debian-router
ssh debian-client
```

接続経路：

```text
MacBook
   ↓
Windows
   ↓
debian-router
   ↓
debian-client
```

したがって、外出先のMacBookからRouter VMとClient VMの両方をSSHで操作できる状態になっている。

---

### 将来的な公開鍵認証

現在はパスワード認証でもよいが、運用が安定した後はMacBookのSSH公開鍵を各マシンへ登録し、

```text
MacBook
 ↓
Windows
 ↓
Router
 ↓
Client
```

を公開鍵認証で接続できるようにする。

これにより、外出先からの接続がより簡単になる。

---


## 10.4 LAN側IPの一時設定と疎通確認

Router VMとClient VMをVMnet2へ接続し、LAN側IPを一時的に設定した。

設定値：

```text
Debian Router VM
LAN: 10.0.0.1/24

Debian Client VM
LAN: 10.0.0.10/24
```

一時設定には `ip` コマンドを使用した。

Router側の例：

```bash
sudo ip link set <LAN_NIC> up
sudo ip addr add 10.0.0.1/24 dev <LAN_NIC>
```

Client側の例：

```bash
sudo ip link set <CLIENT_NIC> up
sudo ip addr add 10.0.0.10/24 dev <CLIENT_NIC>
```

この状態で、

```text
Router 10.0.0.1
    │
    │ VMnet2
    │
Client 10.0.0.10
```

の相互通信を確認した。

ClientからRouter：

```bash
ping 10.0.0.1
```

RouterからClient：

```bash
ping 10.0.0.10
```

どちらも成功し、**VMnet2上でRouter VMとClient VMが相互通信できることを確認済み**。

### 現在の注意点

今回のIP設定は、

```bash
ip addr add ...
```

による一時設定である。

そのため、

```text
一時設定
   ↓
VM再起動
   ↓
IP設定が消える
```

という状態。

永続化は次回作業として、

```text
/etc/network/interfaces
```

などのDebianのネットワーク設定へ記述する予定。

現時点では、

```text
一時設定で疎通確認
        ↓
仕組みを理解
        ↓
永続設定へ移行
```

という順序で進めている。

---

# 11. TODO

このプロジェクトではTODOリストを用意する。

理由：

- 環境構築
- ネットワーク学習
- Router Software開発
- 実機移植

まで工程が長く、現在位置を把握しやすくするため。

## Phase 1 - Virtual Lab Setup

- [x] VMware Workstation ProをWindowsへインストール
- [x] Debian ISOをダウンロード
- [x] Debian Router VMを作成
- [x] Debian Router VMへDebianをインストール
- [x] Debian Client VMを作成
- [x] Debian Client VMへDebianをインストール
- [x] MacBookからWindowsへTailscale経由でSSH接続
- [x] WindowsからRouter VMへSSH接続
- [x] MacBookからWindows経由でRouter VMへSSH接続
- [x] LAN用VMnet2を作成
- [x] Router VMへ2枚目の仮想NICを追加
- [x] Router VMの2枚目NICをVMnet2へ接続
- [x] Client VMのNICをVMnet2へ変更
- [x] Router VM / Client VM双方でNICを確認
- [ ] Router VMからClient VMへSSH接続確認
- [ ] MacBookの `~/.ssh/config` にWindows / Router / Clientを登録
- [x] `ssh debian-router` でRouter VMへ接続確認
- [x] `ssh debian-client` でClient VMへ多段SSH接続確認

## Phase 2 - Basic Network Setup

- [x] Router LAN側に `10.0.0.1/24` を一時設定
- [x] Clientに `10.0.0.10/24` を一時設定
- [ ] Router / ClientのIP設定を `/etc/network/interfaces` などへ記述して永続化
- [ ] ClientのDefault Gatewayを `10.0.0.1` に設定
- [x] Router ↔ Client間で相互 `ping` 確認
- [ ] `ip addr` を理解
- [ ] `ip link` を理解
- [ ] `ip route` を理解
- [ ] `ip neigh` を理解
- [ ] `tcpdump` でPacketを観察

## Phase 3 - Router Fundamentals

- [ ] IP Forwardingを学習
- [ ] IP Forwardingを有効化
- [ ] Routing Tableを学習
- [ ] Client → Router → WAN のPacket経路を確認
- [ ] NATを学習
- [ ] nftablesでNATを設定
- [ ] ClientからInternetへの疎通確認
- [ ] Firewallを学習
- [ ] nftablesでFirewall Ruleを設定

## Phase 4 - Network Services

- [ ] DHCPの仕組みを学習
- [ ] DHCP Serverを構築
- [ ] ClientへIPを自動配布
- [ ] DNSの仕組みを学習
- [ ] DNS Forwarderを構築
- [ ] IPv6を学習
- [ ] VLANを学習
- [ ] VPNを学習

## Phase 5 - Automation

- [ ] 手動設定を整理
- [ ] 設定ファイルを `configs/` に保存
- [ ] Setup Scriptを作成
- [ ] Test Scriptを作成
- [ ] Reset / Cleanup Scriptを作成

## Phase 6 - Router Software

- [ ] `routerd` の責務を決定
- [ ] Interface Management
- [ ] Routing Management
- [ ] Firewall Management
- [ ] NAT Management
- [ ] Traffic Monitoring
- [ ] CLI / API
- [ ] NetlinkによるLinux Kernel制御を検討

## Phase 7 - Hardware Selection

- [ ] CPU使用量測定
- [ ] Memory使用量測定
- [ ] Network負荷測定
- [ ] 必要NIC数決定
- [ ] 1GbE / 2.5GbE選定
- [ ] ARM64 / x86-64選定
- [ ] SBC / Mini PC比較
- [ ] 実機購入

## Phase 8 - Physical Router Deployment

- [ ] 実機へDebianをインストール
- [ ] `routerd` をビルド
- [ ] `configs/` を実機へ反映
- [ ] `deploy/install.sh` を作成
- [ ] systemd Service化
- [ ] 実WAN NIC接続
- [ ] 実LAN NIC接続
- [ ] Routing確認
- [ ] NAT確認
- [ ] Firewall確認
- [ ] 実環境での安定動作確認

---

## 12. 次にやること

現在、VMnet2上で以下の一時設定まで完了している。

```text
Router LAN : 10.0.0.1/24
Client     : 10.0.0.10/24
```

Router VMとClient VMの相互Pingも成功済み。

次回はまず、

> **Router / ClientのIP設定を永続化する**

ことから進める。

候補はDebianの、

```text
/etc/network/interfaces
```

への設定記述。

永続化後にVMを再起動し、

```bash
ip -br addr
ip route
ping
```

で設定が保持されていることを確認する。

その後、

1. `ip route`
2. `ip neigh`
3. `tcpdump`
4. ClientのDefault Gateway設定
5. IP Forwarding
6. NAT

の順で、本格的なルーター学習へ進む。
