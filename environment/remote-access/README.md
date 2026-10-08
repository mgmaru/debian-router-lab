# Remote Access

外出先のMacBookから、Windows上のRouter VM / Client VMをSSHで操作するための設定を記録する。

この構成を採用した理由は [005-ssh-via-jump-host.md](../../docs/decisions/005-ssh-via-jump-host.md) を参照。

## 構成

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

## Windowsへの接続

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

## Router VMへの接続

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

## Client VMへの接続

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

## MacBookのSSH Config

外出先のMacBookから毎回長いSSHコマンドを入力しなくて済むように、`~/.ssh/config` に接続先を登録する。

MacBookで以下を開く。

```bash
nano ~/.ssh/config
```

またはVS Codeなどのエディタで `~/.ssh/config` を編集する。

### 設定例

設定例は [ssh_config.example](ssh_config.example) にある。

以下のIPアドレスは環境に合わせて置き換える。

- `100.80.124.76` : WindowsのTailscale IP
- `192.168.x.x` : Debian Router VMのWAN側IP
- `10.0.0.10` : Debian Client VMのLAN側IP

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
