# 01 - Network Interface

## Goal

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

## Topology

```
                         ┌──────────── ルーター VM (debian-router) ────────────┐
                         │                                                     │
クライアント VM ─────────┤ ens37: 10.0.0.1/24        ens33: 192.168.22.129/24 ├──── VMware NAT ──── インターネット
ens33: 10.0.0.10/24      │ （内側 = LAN）            （外側 = WAN、DHCP）      │
                         └─────────────────────────────────────────────────────┘
```

Router VMのens37とClient VMのens33は、VMnet2で接続している。

各インターフェースの設定は [environment/debian/network.md](../../environment/debian/network.md#構成) を参照。

## Experiment

### 1. LAN側IPの一時設定と疎通確認

Router VMとClient VMをVMnet2へ接続し、LAN側IPを一時的に設定した。

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

この状態で、Router VMとClient VMの相互通信を確認した。

ClientからRouter：

```bash
ping 10.0.0.1
```

RouterからClient：

```bash
ping 10.0.0.10
```

### 2. IP設定の永続化

`ip addr add` で付けたIPアドレスは、**再起動すると消えてしまう**。
これを、Debianの設定ファイル `/etc/network/interfaces` に書くことで、**再起動してもIPが残る（＝永続化する）**ようにした。

あわせて、ルーターを作るうえで欠かせない**デフォルトゲートウェイ**の考え方も整理した（[routing.md](../../docs/concepts/routing.md)）。

手順は [environment/debian/network.md](../../environment/debian/network.md)、設定ファイルは [configs/network](../../configs/network/README.md) を参照。

#### 作業前の状態の観察

Router VMで `ip a` を実行すると、ens37は次のように表示された。

```
3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ... state UP ...
    link/ether 00:0c:29:73:2d:6b brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/24 scope global ens37
       valid_lft forever preferred_lft forever
```

`dynamic` が付いておらず `valid_lft forever` なので、**ens37の `10.0.0.1` は手で付けた（＝再起動で消える）アドレス**だとわかる（読み方は [interface.md](../../docs/concepts/interface.md)）。

Client VMでは次のように表示された。

```
2: ens33: ...
    inet 169.254.165.204/16 brd 169.254.255.255 scope global noprefixroute ens33
    inet 10.0.0.10/24 scope global ens33
```

`169.254.165.204` はリンクローカルアドレスである。
このネットワーク（10.0.0.0/24）にはまだDHCPサーバーがいない（Router VMはIPを持っているだけで、配ってはいない）ため、DHCPクライアントが問い合わせに失敗し、代わりにこのアドレスを付けていた。
害はないが不要なので、静的IPに切り替えれば再起動後には付かなくなる。

#### 行ったこと

| VM | 変更内容 |
|---|---|
| Router VM | ens33（WAN）はDHCPのまま、ens37（LAN）に `10.0.0.1/24` を静的に設定 |
| Client VM | ens33を `dhcp` から `static` に書き換え、`10.0.0.10/24` とゲートウェイ `10.0.0.1` を設定 |

### 3. デフォルトゲートウェイの効果を確かめる

ゲートウェイの意味を体感するための実験（Client VMで実行）。

**ゲートウェイがない状態で：**

```bash
ping -c 3 192.168.22.129     # ルーター VM の「外側」の IP
```

→ `Network is unreachable` になるはずである。`192.168.22.129` は自分のネットワーク（10.0.0.0/24）の外なので、渡す先がない。

**一時的にゲートウェイを設定してから：**

```bash
sudo ip route add default via 10.0.0.1
ping -c 3 192.168.22.129
```

→ 応答が返ってくるはずである。Client VMが「範囲外だから `10.0.0.1` に渡す」と判断し、Router VMが「自分のアドレス宛てだ」と受け取って返事をした、という流れである。

（`ip route add` は一時的な設定なので、再起動すれば消える。安心して試せる）

### 4. ルーティングテーブルの観察（Router VM）

Router VMで `ip route` を実行し、ルーティングテーブルを確認した（2026-10-11）。

```bash
ip route
```

```text
default via 192.168.22.2 dev ens33 proto dhcp src 192.168.22.129 metric 1002
10.0.0.0/24 dev ens37 proto kernel scope link src 10.0.0.1
192.168.22.0/24 dev ens33 proto dhcp scope link src 192.168.22.129 metric 1002
```

ルーティングテーブルの考え方は [what-is-ip-route-and-routing-table.md](../../docs/questions/2026-10-11-what-is-ip-route-and-routing-table.md) を参照。

### 5. 近隣テーブルの観察（Router VM / Client VM）

Router VMとClient VMで `ip neigh` を実行し、近隣テーブルを確認した（2026-10-11）。

```bash
ip neigh
```

Router VM：

```text
kajih@debian-router:~$ ip neigh
192.168.22.2 dev ens33 lladdr 00:50:56:f9:12:3d STALE
192.168.22.1 dev ens33 lladdr 00:50:56:c0:00:08 REACHABLE
10.0.0.10 dev ens37 lladdr 00:0c:29:20:6c:ff STALE
192.168.22.254 dev ens33 lladdr 00:50:56:fe:ec:fe STALE
```

Client VM：

```text
kajih@debian-client:/etc/network$ ip neigh
10.0.0.1 dev ens33 lladdr 00:0c:29:73:2d:6b DELAY
```

## Result

### 1. LAN側IPの一時設定と疎通確認

どちらも成功し、**VMnet2上でRouter VMとClient VMが相互通信できることを確認済み**。

ただし `ip addr add` による一時設定のため、VMを再起動するとIP設定が消える状態だった。

```text
一時設定で疎通確認
        ↓
仕組みを理解
        ↓
永続設定へ移行
```

という順序で進め、永続化は Experiment 2 で行った。

### 2. IP設定の永続化

`/etc/network/interfaces` に設定を書き、再起動してもIPアドレスが残ることを確認した。

### 4. ルーティングテーブルの観察（Router VM）

ルールが3行入っていた。

| 行 | 一文に訳すと | 誰が作ったか |
|---|---|---|
| `default via 192.168.22.2 dev ens33 ...` | どの行にも当てはまらない宛先は、ens33から出してVMware NAT（`192.168.22.2`）に渡す | DHCP（`proto dhcp`） |
| `10.0.0.0/24 dev ens37 ...` | `10.0.0.0/24`（LAN）宛ては、ens37から直接届ける | カーネル（`proto kernel`） |
| `192.168.22.0/24 dev ens33 ...` | `192.168.22.0/24`（WAN）宛ては、ens33から直接届ける | DHCP（`proto dhcp`） |

- NICが2つあるので、「自分のネットワーク」の行も2つある
- デフォルトゲートウェイはens33（WAN側）の1つだけで、ens37には無い（ルーターの内側には `gateway` を書かない、の通り）
- `metric 1002` は、DHCPクライアント（dhcpcd）が付けた優先度（1000 ＋ インターフェース番号 `2`）。`default` は1行しかないので、今は影響しない

### 5. 近隣テーブルの観察（Router VM / Client VM）

`ip neigh` は、**同じネットワークにいる相手の、IPアドレスとMACアドレスの対応表**（近隣テーブル。IPv4ではARPテーブルとも呼ぶ）を表示する。

1行の読み方と状態（`REACHABLE` / `STALE` / `DELAY` など）の意味は [neighbor.md](../../docs/concepts/neighbor.md#ip-neigh-の読み方) を参照。

Router VMの相手：

| IP | NIC | 相手 | 状態 |
|---|---|---|---|
| `192.168.22.2` | ens33 | VMware NAT（Router VMのデフォルトゲートウェイ） | `STALE` |
| `192.168.22.1` | ens33 | Windows（VMware Network Adapter VMnet8）。SSHの接続元 | `REACHABLE` |
| `10.0.0.10` | ens37 | Client VM | `STALE` |
| `192.168.22.254` | ens33 | VMwareのDHCPサーバー（推定。VMwareのNATネットワークでは通常 `.254`） | `STALE` |

Client VMの相手：

| IP | NIC | 相手 | 状態 |
|---|---|---|---|
| `10.0.0.1` | ens33 | Router VMのens37（Client VMのデフォルトゲートウェイ） | `DELAY` |

ルーティングテーブルとの関係：

```text
宛先 8.8.8.8
    ↓ ルーティングテーブル（ip route）
次に渡す相手 = 192.168.22.2（ens33 から）
    ↓ 近隣テーブル（ip neigh）
192.168.22.2 の MAC = 00:50:56:f9:12:3d
    ↓
その MAC 宛てに送る
```

- Client VMから見た `10.0.0.1` のMAC（`00:0c:29:73:2d:6b`）は、Router VMのens37のMAC（Experiment 2の `ip a`）と一致した。デフォルトゲートウェイに渡すときは、実際にはこのMAC宛てに送っている
- 近隣テーブルに載るのは、**同じネットワークにいて、最近通信した相手だけ**。`8.8.8.8` のような外の宛先は載らない
- `192.168.22.1` だけが `REACHABLE` なのは、WindowsからのSSHで今まさに通信しているから
- Client VMの `DELAY` は、Router VM経由のSSHの通信で、ちょうど使われたところだったため（推定）
- MACアドレスの先頭 `00:0c:29` はVMwareのVMのNIC、`00:50:56` はVMwareが用意した仮想機器（NAT・DHCP・Windows側のアダプタ）。`00:50:56:c0:00:08` の末尾 `08` はVMnet8を表す

## Learned

- `ip` コマンドの設定は**カーネルの中だけ**の変更で、再起動で消える
- 永続化 ＝ **起動時に読まれる設定ファイルに書く**こと（Debian の ifupdown なら `/etc/network/interfaces`）
- まず **誰がその NIC を管理しているか**を確認し、管理を 1 つに絞る
- `/etc/network/interfaces` は **1 行 1 指示**。字下げは任意、全角スペースは NG
- **デフォルトゲートウェイ** ＝「自分のネットワーク外の宛先を全部渡す窓口」。クライアントには書き、ルーターの内側には書かない
- **DHCP は IP アドレスだけでなく、サブネットマスク・デフォルトゲートウェイ・DNS もまとめて配る**。ルーター VM の ens33 は DHCP なので、ゲートウェイを自分で書かなくても手に入っていた（不要なのではなく、自動で入っている）
- 反映は `ifdown`（書き換え前）→ `ifup`、**最後に必ず再起動して確認**
- ネットワーク設定の変更は「**今使っている接続を切らない**」意識で。困ったら VM のコンソールから直す
- **ルーティングテーブル** ＝「宛先ごとの送り方」の一覧。ルーター VM では NIC ごとに自分のネットワークの行があり、`default` は WAN 側（ens33）にだけある
- **近隣テーブル（`ip neigh`）** ＝ 同じネットワークの相手の IP と MAC の対応表。ルーティングテーブルで決めた「次に渡す相手」の IP を、ここで MAC に変換して送る
