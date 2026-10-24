# 02 - IP Forwarding

## Goal

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

## Notes

Linux は既定では、自分宛てでないパケットを受け取っても捨てる。これを中継するように変える。

```bash
# 一時的
sudo sysctl -w net.ipv4.ip_forward=1

# 永続化（起動時に読まれる場所に書く）
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-router.conf
sudo sysctl --system
```

> 💡 Debian 13 では `/etc/sysctl.conf` が標準では存在しないので、`/etc/sysctl.d/` に書くのが確実。

永続化の考え方は [persistent-config.md](../../docs/concepts/persistent-config.md) を参照。

## Topology

```text
Client VM                       Router VM                                 VMware NAT
10.0.0.10  ──→  ens37（10.0.0.1）        ens33（192.168.22.129）  ──→  192.168.22.2
                     │                        ↑
                     └── 自分宛てではない ──┘
                         ip_forward=0：ここで捨てる
                         ip_forward=1：ens33 から送り出す
```

Client VMからRouter VMの**向こう側**（VMware NAT の `192.168.22.2`）へ `ping` を打ち、Router VMが転送するかどうかを、両側のNICの `tcpdump` で観察する。

## Experiment

### 1. 現在の設定の確認（Router VM）

転送の設定（カーネルパラメータ `net.ipv4.ip_forward`）を確認した（2026-10-24）。

```text
kajih@debian-router:~$ sysctl net.ipv4.ip_forward
-bash: sysctl: command not found
kajih@debian-router:~$ sudo sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0
```

### 2. 転送が無効のまま `ping` を打つ

Router VMで、ens37（LAN側）とens33（WAN側）の `tcpdump` をそれぞれ別のターミナルで動かしたまま、Client VMから `192.168.22.2` へ `ping` を打った（2026-10-24）。
ens37 には `-e` を付けて、MACアドレスも表示した。

Router VM ens37（LAN側）：

```text
kajih@debian-router:~$ sudo tcpdump -i ens37 -n -e icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens37, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21:59:07.689162 00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: 10.0.0.10 > 192.168.22.2: ICMP echo request, id 3, seq 1, length 64
21:59:08.693437 00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: 10.0.0.10 > 192.168.22.2: ICMP echo request, id 3, seq 2, length 64
21:59:09.717452 00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: 10.0.0.10 > 192.168.22.2: ICMP echo request, id 3, seq 3, length 64
```

Router VM ens33（WAN側）：

```text
kajih@debian-router:~$ sudo tcpdump -i ens33 -n icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens33, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

Client VM：

```text
kajih@debian-client:~$ ping -c 3 192.168.22.2
PING 192.168.22.2 (192.168.22.2) 56(84) bytes of data.

--- 192.168.22.2 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2028ms
```

### 3. 転送を一時的に有効にして、もう一度 `ping` を打つ

`tcpdump` は動かしたまま、Router VMで転送を有効にし、Client VMから同じ `ping` を打った（2026-10-24）。

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Router VM ens37（LAN側）：

```text
22:21:05.906334 00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: 10.0.0.10 > 192.168.22.2: ICMP echo request, id 4, seq 1, length 64
22:21:06.923096 00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: 10.0.0.10 > 192.168.22.2: ICMP echo request, id 4, seq 2, length 64
22:21:07.946739 00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: 10.0.0.10 > 192.168.22.2: ICMP echo request, id 4, seq 3, length 64
```

Router VM ens33（WAN側）：

```text
kajih@debian-router:~$ sudo tcpdump -i ens33 -n icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens33, link-type EN10MB (Ethernet), snapshot length 262144 bytes
22:21:05.906360 IP 10.0.0.10 > 192.168.22.2: ICMP echo request, id 4, seq 1, length 64
22:21:06.923112 IP 10.0.0.10 > 192.168.22.2: ICMP echo request, id 4, seq 2, length 64
22:21:07.946750 IP 10.0.0.10 > 192.168.22.2: ICMP echo request, id 4, seq 3, length 64
```

Client VM：

```text
kajih@debian-client:~$ ping -c 3 192.168.22.2
PING 192.168.22.2 (192.168.22.2) 56(84) bytes of data.

--- 192.168.22.2 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2040ms
```

### 4. 元に戻す（Router VM）

転送を無効に戻した（2026-10-24）。`sysctl -w` は一時的な設定なので、再起動しても `0` に戻る。

```bash
sudo sysctl -w net.ipv4.ip_forward=0
```

## Result

### 1. 現在の設定の確認（Router VM）

- `net.ipv4.ip_forward = 0`。Linux の初期状態のままで、**転送は無効**だった
- 一般ユーザーで `sysctl` を打つと `command not found` になった。`sysctl` は `/usr/sbin/sysctl` にあり、Debian では `/usr/sbin` が一般ユーザーの `PATH` に入っていないため。`sudo sysctl`、`/usr/sbin/sysctl`、`cat /proc/sys/net/ipv4/ip_forward` のどれでも読める
- `sysctl` は、動いているカーネルの設定値（カーネルパラメータ）を読み書きするコマンド。名前の `.` を `/` に置き換えると、`/proc/sys/` の下のファイルの場所になる（`net.ipv4.ip_forward` → `/proc/sys/net/ipv4/ip_forward`）。サービスを操作する `systemctl` とは別物

### 2. 転送が無効のまま `ping` を打つ

| 観察した場所 | 結果 | 分かったこと |
|---|---|---|
| ens37（LAN側） | echo request が3つ届いた | Client VMは、パケットをRouter VMに渡している |
| ens33（WAN側） | **何も出なかった** | Router VMが、受け取ったパケットを**捨てている** |
| Client VM | `100% packet loss`。エラーも出ない | Router VMは、捨てたことを知らせない（ICMPを返さない） |

ens37 の1行（`-e` 付き）の読み方：

```text
00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: 10.0.0.10 > 192.168.22.2: ICMP echo request ...
```

| 部分 | 意味 |
|---|---|
| `00:0c:29:20:6c:ff >` | MACの送信元：Client VMのens33 |
| `> 00:0c:29:73:2d:6b` | **MACの宛先：Router VMのens37** |
| `ethertype IPv4 (0x0800)` | Ethernetの中身はIPv4 |
| `length 98` | Ethernet全体の長さ。Ethernetヘッダ14バイト ＋ IPパケット84バイト（`ping` の `56(84)` の84） |
| `10.0.0.10 > 192.168.22.2` | **IPの宛先：VMware NAT**（Router VMではない） |

パケットには宛先が2つある。

| 宛先 | 中身 | 意味 |
|---|---|---|
| IPの宛先（最終的な届け先） | `192.168.22.2` | VMware NAT。Router VMではない |
| MACの宛先（次に渡す相手） | `00:0c:29:73:2d:6b` | Router VMのens37 |

Client VMは、`192.168.22.2` が自分のネットワーク（`10.0.0.0/24`）の外なので、ルーティングテーブルに従ってデフォルトゲートウェイ（`10.0.0.1`）に渡す。近隣テーブルで `10.0.0.1` のMACを調べ、そのMAC宛てに送っている（labs/01 の Experiment 4・5 の流れ）。

Router VMから見ると、「**自分に渡されたが、自分宛てではない**」パケットである。`ip_forward = 0` なので、捨てた。

### 3. 転送を一時的に有効にして、もう一度 `ping` を打つ

| 観察した場所 | 結果 | 分かったこと |
|---|---|---|
| ens37（LAN側） | echo request が3つ届いた | 2回目と同じ |
| ens33（WAN側） | **echo request が3つ出ていった** | Router VMが**転送した** |
| ens33（WAN側） | echo reply は来なかった | VMware NATから返事が戻らない（帰り道の問題） |
| Client VM | `100% packet loss` | 返事が戻らないので、転送が有効でも `ping` は失敗する |

同じパケットが、ens37 から ens33 へ移っている（`id 4` と `seq` が一致）。

| seq | ens37 に届いた時刻 | ens33 から出た時刻 | 差 |
|---|---|---|---|
| 1 | `22:21:05.906334` | `22:21:05.906360` | 約26マイクロ秒 |
| 2 | `22:21:06.923096` | `22:21:06.923112` | 約16マイクロ秒 |
| 3 | `22:21:07.946739` | `22:21:07.946750` | 約11マイクロ秒 |

- Router VMは、受け取ってから数十マイクロ秒で転送している
- ens33 から出ていくパケットも、送信元は `10.0.0.10` のまま。Router VMは**中継しただけで、IPアドレスは書き換えていない**
- どのNICから出すかは、ルーティングテーブルで決まる。宛先 `192.168.22.2` は `192.168.22.0/24 dev ens33` の行に当てはまるので、ens33 から出た

#### 返事が戻らない理由：帰り道

```text
行き：Client VM → Router VM → VMware NAT            ← 届いている
帰り：VMware NAT → 10.0.0.10 への返し方を知らない   ← 戻ってこない
```

VMware NATが知っているのは、自分のネットワーク（`192.168.22.0/24`）だけである。「`10.0.0.0/24` はRouter VM（`192.168.22.129`）の向こうにある」という経路を持っていないので、返事をRouter VMに返せない。

これは NAT（labs/04）で解決する。送信元をRouter VMの `192.168.22.129` に書き換えれば、VMware NATはRouter VMに返事を送れる。

#### 見えていないが、起きていること

Router VMは、転送するときにパケットを2か所書き換えている。今回の表示には出ていない。

| 書き換え | 確かめ方 |
|---|---|
| **MACアドレス**：送信元がRouter VMのens33、宛先がVMware NATのMACに変わる | ens33 側の `tcpdump` に `-e` を付ける |
| **TTLが1減る**：ルーターを1つ通るごとに1減る（64 → 63） | 両側の `tcpdump` に `-v` を付けて、`ttl` を比べる |

IPアドレスはそのままで、MACアドレスは区間ごとに付け替えられる。これがルーターの基本的な動きである。

#### ルーターの役割の分担

```text
転送してよいか     → ip_forward（labs/02）
どこへ送るか       → ルーティングテーブル（labs/03）
通してよいか       → ファイアウォール（labs/05）
```

`net.ipv4.ip_forward` はRouter VM全体のスイッチで、「ens33 と ens37 の間だけ」のような指定はできない。通す・通さないを細かく決めるのは、ファイアウォールの役目である。

## Learned

- **IP Forwarding** ＝ 自分宛てではないパケットを受け取ったときに、捨てずに別のNICから送り出すことを許可する、カーネルの設定（`net.ipv4.ip_forward`）。Linux の初期状態は `0`（無効）
- 転送が無効のルーターは、自分宛てではないパケットを**黙って捨てる**。送った側には、エラーも返らない
- パケットには、**IPの宛先**（最終的な届け先）と**MACの宛先**（次に渡す相手）がある。ルーターには、「MACの宛先は自分、IPの宛先は自分ではない」パケットが届く
- 転送しても、**IPアドレスは書き換わらない**。書き換わるのはMACアドレスとTTL
- 転送できても、**帰り道**が無いと返事は戻らない。相手が送信元（`10.0.0.0/24`）への返し方を知らないため。NATで解決する
- `sysctl` はカーネルの設定値を読み書きするコマンド（`/proc/sys/` の下のファイル）。Debian の一般ユーザーでは `PATH` に無いので、`sudo` を付ける
