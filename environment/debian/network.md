# Network Setup

Router VM / Client VM の IP アドレスを、`/etc/network/interfaces` に書いて永続化する手順。

- 永続化の考え方：[docs/concepts/persistent-config.md](../../docs/concepts/persistent-config.md)
- 設定ファイルと書き方：[configs/network](../../configs/network/README.md)
- ifupdown を使う理由：[006-use-ifupdown.md](../../docs/decisions/006-use-ifupdown.md)
- SSH 越しに作業するときの注意：[environment/remote-access](../remote-access/README.md#ssh-越しにネットワーク設定を変更するとき)

---

## 構成

```
                         ┌──────────── ルーター VM (debian-router) ────────────┐
                         │                                                     │
クライアント VM ─────────┤ ens37: 10.0.0.1/24        ens33: 192.168.22.129/24 ├──── VMware NAT ──── インターネット
ens33: 10.0.0.10/24      │ （内側 = LAN）            （外側 = WAN、DHCP）      │
                         └─────────────────────────────────────────────────────┘
```

| VM | インターフェース | IP アドレス | 取得方法 | 役割 |
|---|---|---|---|---|
| ルーター VM | ens33 | 192.168.22.129/24 | DHCP（VMware がくれる） | 外側（WAN） |
| ルーター VM | ens37 | 10.0.0.1/24 | **静的** | 内側（LAN） |
| クライアント VM | ens33 | 10.0.0.10/24 | **静的** | LAN 内の端末 |

> 💡 ルーター VM の ens37 とクライアント VM の ens33 は、VMware 上で **同じ仮想ネットワーク（同じ VMnet / LAN セグメント）** につながっている必要があります。ここがずれていると、IP を正しく設定しても通信できません。

---

## 誰がネットワークを管理しているかを確認する

Debian でネットワークを管理する仕組みは主に 3 つあり、**どれが使われているかで、書く場所が変わります**（[006-use-ifupdown.md](../../docs/decisions/006-use-ifupdown.md) を参照）。

### 確認方法

```bash
systemctl is-active networking NetworkManager systemd-networkd
```

今回は `networking` が `active` でした。

### ⚠️ ただし `active` だけでは判断しきれない

`networking` サービスは、`lo`（ループバック）しか設定していなくても `active` と表示されます。**確実なのは設定ファイルの中身を見ること**です。

```bash
cat /etc/network/interfaces
```

今回のファイル（インストール直後の状態）：

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug ens33
iface ens33 inet dhcp
```

ens33 がここに `dhcp` で書かれているので、**ens33 は ifupdown が管理している**ことが確定します。
（ens37 は VM 作成後に追加した NIC なので、ここには書かれていません）

> 💡 **同じ NIC を複数の仕組みで管理しないこと**が鉄則です。設定が取り合いになって、意図しない IP になったり、IP が消えたりします。なお Debian の NetworkManager は、`/etc/network/interfaces` に書かれた NIC には手を出さない設定になっています。

---

## 手順：ルーター VM

ルーター VM では、**ens33（外側）は DHCP のまま触らず、ens37（内側）を末尾に追加**します。

### ① ホスト名で、ルーター VM にいることを確認

プロンプトが `kajih@debian-router:~$` のようになっていることを確認します。2 台とも `/etc/network/interfaces` の初期状態は同じなので、**どちらの VM を操作しているかの確認は必須**です。

### ② バックアップを取る

```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.bak
```

間違えても `sudo cp /etc/network/interfaces.bak /etc/network/interfaces` で元に戻せます。

### ③ 編集する

```bash
sudo nano /etc/network/interfaces
```

末尾に次を追加します。

```
auto ens37
iface ens37 inet static
    address 10.0.0.1/24
```

`gateway` は書きません。ens33 の DHCP がデフォルトゲートウェイを配ってくれているからです（詳しくは [routing.md](../../docs/concepts/routing.md) と [dhcp.md](../../docs/concepts/dhcp.md)）。

編集後のファイル全体は [configs/network/debian-router.interfaces](../../configs/network/debian-router.interfaces) を参照。

### ④ 文法チェック

```bash
sudo ifup --no-act ens37
```

### ⑤ 反映する

ens37 は今まで ifupdown の管理外だったので、手で付けた IP を消してから上げます。

```bash
sudo ip addr flush dev ens37   # 手で付けた 10.0.0.1 を消す
sudo ifup ens37                # 設定ファイルの内容で上げ直す
```

> 💡 先に `flush` するのは、同じ IP がすでに付いていると `ifup` が「もうある（File exists）」とエラーで止まるからです。

### ⑥ 再起動して永続化を確認

```bash
sudo reboot
```

再起動後、SSH で入り直して確認します。

```bash
ip a show ens37        # 10.0.0.1/24 が残っていれば OK
```

---

## 手順：クライアント VM

クライアント VM では、**ens33 の設定を `dhcp` から `static` に書き換えます**。

### ① ホスト名で、クライアント VM にいることを確認

### ② バックアップを取る

```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.bak
```

### ③ 書き換える前に、DHCP のまま止める（重要）

```bash
sudo ifdown ens33
```

> ⚠️ **順番が大事です。** `ifdown` は「**今の設定ファイルの内容**」を見て止め方を決めます。ファイルを `static` に書き換えた後に `ifdown` すると、ifupdown は「静的 IP を外せばいい」と判断してしまい、裏で動いている DHCP クライアントがきちんと止まりません。
> もう書き換えてしまった場合は、**再起動するのがいちばん確実**です。

> ⚠️ この VM に ens33 経由で SSH していると、ここで接続が切れます。クライアント VM は **VMware のコンソール画面から作業**してください。

### ④ 書き換える

```bash
sudo nano /etc/network/interfaces
```

最後の 2 行を：

```
allow-hotplug ens33
iface ens33 inet dhcp
```

次のように置き換えます。

```
auto ens33
iface ens33 inet static
    address 10.0.0.10/24
    gateway 10.0.0.1
```

`gateway 10.0.0.1` は「外への出口はルーター VM」という指定です。

編集後のファイル全体は [configs/network/debian-client.interfaces](../../configs/network/debian-client.interfaces) を参照。

### ⑤ 文法チェック

```bash
sudo ifup --no-act ens33
```

### ⑥ 手で付けた IP を消して、新しい設定で上げる

```bash
sudo ip addr flush dev ens33   # 手で付けた 10.0.0.10 や 169.254.x.x を消す
sudo ifup ens33
```

### ⑦ 再起動して永続化を確認

```bash
sudo reboot
```

```bash
ip a show ens33        # 10.0.0.10/24 だけになっていれば OK（169.254 は消えている）
ip route               # default via 10.0.0.1 dev ens33 があれば OK
ping -c 3 10.0.0.1     # ルーター VM に届けば OK
```

---

## 設定を反映する方法

設定ファイルを書いただけでは、今動いているカーネルには反映されません。反映方法は 2 つあります。

| 方法 | コマンド | 特徴 |
|---|---|---|
| インターフェース単位で上げ直す | `sudo ifdown <名前>` / `sudo ifup <名前>` | 他の NIC に影響しない。順番に注意（下記） |
| 再起動する | `sudo reboot` | 確実。永続化の確認も兼ねられる |

### `ifdown` / `ifup` を使うときのルール

1. **設定を書き換える前に** `ifdown`（`ifdown` は今のファイルの内容を見て止め方を決めるため）
2. 手で付けた IP が残っていたら `ip addr flush dev <名前>` で消す
3. 書き換えた後に `ifup`

### 結論：最後に一度は再起動する

目的は「**再起動しても消えないこと**」です。`ifup` で反映できても、本当に永続化できたかは再起動してみないとわかりません。

**`ifup` で動作確認 → 最後に再起動して `ip a` で確認** という流れにすると安心です。

---

## 動作確認

| 確認したいこと | コマンド | 期待する結果 |
|---|---|---|
| IP が付いているか | `ip a show <名前>` | `inet 10.0.0.x/24` があり、`dynamic` が付いていない |
| デフォルトゲートウェイ | `ip route` | （クライアント VM）`default via 10.0.0.1 dev ens33`<br>（ルーター VM）`default via 192.168.22.2 dev ens33`（DHCP で自動設定） |
| DHCP で受け取った内容 | `sudo dhcpcd -U ens33` | （ルーター VM）`routers` にゲートウェイが入っている |
| ルーター VM に届くか | `ping -c 3 10.0.0.1` | 応答が返ってくる |
| 設定ファイルの文法 | `sudo ifup --no-act <名前>` | エラーが出ない |
| networking サービスのログ | `journalctl -u networking -b` | エラーが出ていない |

---

## よくあるトラブル

| 症状 | 考えられる原因 | 対処 |
|---|---|---|
| `ifup` で `RTNETLINK answers: File exists` | 同じ IP がすでに手で付いている | `sudo ip addr flush dev <名前>` してから `ifup` |
| 再起動後に IP が付いていない | `auto <名前>` の行がない／インターフェース名の打ち間違い | `cat /etc/network/interfaces` と `ip a` の名前を見比べる |
| 文法チェックでエラー | 1 行にまとめて書いた／全角スペース | 1 行 1 指示にする。空白を英数入力で打ち直す |
| `169.254.x.x` が残る | `dhcp` の行が残っている | `iface <名前> inet dhcp` を `static` に**書き換え**（追加ではなく） |
| 設定は正しいのに ping が通らない | VM 同士が別の仮想ネットワークにつながっている | VMware の VM 設定で、両方の NIC が同じ VMnet / LAN セグメントか確認 |
| SSH で入れなくなった | ネットワーク設定ミス | VMware のコンソールからログインし、バックアップを戻す |
| 同じ IP 帯なのに不安定 | 同じ NIC を ifupdown と NetworkManager の両方が管理している | 管理を 1 つに絞る（[誰がネットワークを管理しているかを確認する](#誰がネットワークを管理しているかを確認する)） |

---

## コマンド早見表

### 確認系

| コマンド | 内容 |
|---|---|
| `ip a` | IP アドレスの一覧 |
| `ip a show ens37` | 特定のインターフェースだけ |
| `ip route` | 経路表（デフォルトゲートウェイなど） |
| `cat /etc/network/interfaces` | 設定ファイルの中身 |
| `systemctl is-active networking NetworkManager systemd-networkd` | どの管理ツールが動いているか |
| `journalctl -u networking -b` | networking サービスのログ（今回の起動分） |
| `hostname` | どの VM にいるか |

### 一時的な設定（再起動で消える）

| コマンド | 内容 |
|---|---|
| `sudo ip addr add 10.0.0.1/24 dev ens37` | IP を付ける |
| `sudo ip addr flush dev ens37` | そのインターフェースの IP を全部消す |
| `sudo ip link set ens37 up` | インターフェースを上げる |
| `sudo ip route add default via 10.0.0.1` | デフォルトゲートウェイを設定 |

### 永続化・反映

| コマンド | 内容 |
|---|---|
| `sudo cp /etc/network/interfaces /etc/network/interfaces.bak` | バックアップ |
| `sudo nano /etc/network/interfaces` | 設定ファイルを編集 |
| `sudo ifup --no-act ens37` | 文法チェック（実際には何もしない） |
| `sudo ifdown ens37` | インターフェースを止める（**書き換え前に**） |
| `sudo ifup ens37` | 設定ファイルの内容で上げる |
| `sudo reboot` | 再起動（永続化の最終確認） |
