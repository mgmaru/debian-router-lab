# 03 - Routing

## Goal

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

## Notes

`ip route get <宛先>` を使うと、その宛先へのパケットがルーティングテーブルのどの行に当てはまり、どこへ渡されるかを確かめられる。実際にパケットは送らない。

## Topology

```text
Client VM                       Router VM                                 VMware NAT
10.0.0.10  ──→  ens37（10.0.0.1）        ens33（192.168.22.129）  ──→  192.168.22.2  ──→  インターネット（8.8.8.8）
```

Client VMからインターネット上の `8.8.8.8` へ送るパケットが、Client VM → Router VM → WAN と、どの経路で進むかを確かめる。

labs/02 では宛先を `192.168.22.2` にしたので、Router VMは直接つながっているネットワークの行（`192.168.22.0/24 dev ens33`）を使った。今回の宛先は `192.168.22.0/24` の外なので、Router VMでも default の行が使われるはずである。

## Experiment

### 1. Client VMの経路を確認する

Client VMで、`8.8.8.8` へのパケットをどこへ渡すかを調べた（2026-10-25）。

```text
kajih@debian-client:~$ ip route get 8.8.8.8
8.8.8.8 via 10.0.0.1 dev ens33 src 10.0.0.10 uid 1000
    cache
```

### 2. Router VMの経路を確認する

Router VMで、ルーティングテーブル全体と、`8.8.8.8` へのパケットをどこへ渡すかを調べた（2026-10-25）。
Client VMから届いたパケットを転送するときの経路を調べるため、`from`（送信元）と `iif`（受信したNIC）を指定した。比べるため、Router VM自身が送るときの経路も調べた。

```text
kajih@debian-router:~$ ip route
default via 192.168.22.2 dev ens33 proto dhcp src 192.168.22.129 metric 1002
10.0.0.0/24 dev ens37 proto kernel scope link src 10.0.0.1
192.168.22.0/24 dev ens33 proto dhcp scope link src 192.168.22.129 metric 1002
kajih@debian-router:~$ ip route get 8.8.8.8
8.8.8.8 via 192.168.22.2 dev ens33 src 192.168.22.129 uid 1000
    cache
kajih@debian-router:~$ ip route get 8.8.8.8 from 10.0.0.10 iif ens37
8.8.8.8 from 10.0.0.10 via 192.168.22.2 dev ens33
    cache iif ens37
```

### 3. 転送で書き換わるものを観察する（Router VM / Client VM）

Router VMで、ens37（LAN側）とens33（WAN側）の `tcpdump` をそれぞれ別のターミナルで動かしたまま、Client VMから `8.8.8.8` へ `ping -c 3` を打った（2026-10-25）。
`-e` でMACアドレスを、`-v` で `ttl` などのIPヘッダの中身を表示した。

Router VM ens37（LAN側）：

```text
kajih@debian-router:~$ sudo tcpdump -i ens37 -n -e -v icmp
tcpdump: listening on ens37, link-type EN10MB (Ethernet), snapshot length 262144 bytes
19:45:54.018043 00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 18861, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.10 > 8.8.8.8: ICMP echo request, id 6, seq 1, length 64
19:45:55.031053 00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 19106, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.10 > 8.8.8.8: ICMP echo request, id 6, seq 2, length 64
19:45:56.055145 00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 19248, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.10 > 8.8.8.8: ICMP echo request, id 6, seq 3, length 64
```

Router VM ens33（WAN側）：

```text
kajih@debian-router:~$ sudo tcpdump -i ens33 -n -e -v icmp
tcpdump: listening on ens33, link-type EN10MB (Ethernet), snapshot length 262144 bytes
19:45:54.018079 00:0c:29:73:2d:61 > 00:50:56:f9:12:3d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 63, id 18861, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.10 > 8.8.8.8: ICMP echo request, id 6, seq 1, length 64
19:45:55.031062 00:0c:29:73:2d:61 > 00:50:56:f9:12:3d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 63, id 19106, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.10 > 8.8.8.8: ICMP echo request, id 6, seq 2, length 64
19:45:56.055155 00:0c:29:73:2d:61 > 00:50:56:f9:12:3d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 63, id 19248, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.10 > 8.8.8.8: ICMP echo request, id 6, seq 3, length 64
```

## Result

### 1. Client VMの経路を確認する

Client VMは、`8.8.8.8` へのパケットを **`10.0.0.1`（Router VM）に渡し、自分の ens33 から出す**。

| 部分 | 意味 |
|---|---|
| `8.8.8.8` | 調べた宛先 |
| `via 10.0.0.1` | **次に渡す相手（ネクストホップ）**。default の行（`default via 10.0.0.1 dev ens33`）から来ている |
| `dev ens33` | 出すNIC。Client VMのNIC名は `ens33` |
| `src 10.0.0.10` | 送信元に使う自分のIPアドレス |
| `uid 1000` | コマンドを実行したユーザー（kajih）のID。経路の選び方には関係しない |
| `cache` | 調べた結果をカーネルが覚えている、という表示。今は気にしなくてよい |

#### default の行が選ばれた理由

ルーティングテーブルは、宛先に当てはまる行のうち、**いちばん細かい行**（プレフィックスがいちばん長い行）を選ぶ（**ロンゲストマッチ**）。

| 行 | 範囲 | `8.8.8.8` は当てはまるか |
|---|---|---|
| `10.0.0.0/24 dev ens33` | `10.0.0.0`〜`10.0.0.255` | 当てはまらない |
| `default via 10.0.0.1 dev ens33` | すべての宛先（`0.0.0.0/0`） | 当てはまる |

当てはまるのは default の行だけなので、それが選ばれた。default はいちばん粗い行なので、**ほかのどの行にも当てはまらなかったときの行き先**になる。

#### `via 10.0.0.1` で決まるもの

`10.0.0.1` に渡しても、IPの宛先は **`8.8.8.8` のまま**である。`via` で決まるのは、**MACの宛先**（Router VMのens37のMAC）である。labs/02 で見た「IPの宛先は最終的な届け先、MACの宛先は次に渡す相手」と同じ。

### 2. Router VMの経路を確認する

Router VMは、Client VMから届いた `8.8.8.8` へのパケットを **`192.168.22.2`（VMware NAT）に渡し、ens33 から出す**。

ルーティングテーブルの3行（各行の意味は [labs/01 の Result 4](../01-network-interface/README.md#4-ルーティングテーブルの観察router-vm) を参照）のうち、`8.8.8.8` に当てはまるのは default の行だけだった。

| 行 | 範囲 | `8.8.8.8` は当てはまるか |
|---|---|---|
| `10.0.0.0/24 dev ens37` | `10.0.0.0`〜`10.0.0.255`（LAN） | 当てはまらない |
| `192.168.22.0/24 dev ens33` | `192.168.22.0`〜`192.168.22.255`（WAN） | 当てはまらない |
| `default via 192.168.22.2 dev ens33` | すべての宛先（`0.0.0.0/0`） | 当てはまる |

labs/02 の宛先 `192.168.22.2` は `192.168.22.0/24` の行に当てはまったので、ens33 から**直接**届けていた。今回の `8.8.8.8` は default の行に当てはまったので、ens33 から **`192.168.22.2` に渡して**、その先へ送ってもらう。

#### Router VM自身が送る場合と、転送する場合の違い

| 部分 | `ip route get 8.8.8.8`<br>（Router VM自身が送る） | `ip route get 8.8.8.8 from 10.0.0.10 iif ens37`<br>（Client VMのパケットを転送する） |
|---|---|---|
| `via 192.168.22.2` | ある | ある（同じ） |
| `dev ens33` | ある | ある（同じ） |
| 送信元 | `src 192.168.22.129`（Router VMが選んだ自分のIP） | `from 10.0.0.10`（**Client VMのIPのまま**） |
| `uid 1000` | ある | **ない** |
| `iif ens37` | ない | ある |

- **`iif`**（input interface）は、パケットが入ってきたNIC。付けないと、Router VM自身が送るときの経路を調べることになる。`from` だけを付けて `iif` を付けないと、「Router VM自身が送信元 `10.0.0.10` で送る」という意味になり、Router VMは `10.0.0.10` を持っていないのでエラーになる
- **`src` が無い**：`src` は、自分で送るときに送信元に使う自分のIP。転送するパケットは送信元が `10.0.0.10` と決まっているので、Router VMは選ばない。転送してもIPアドレスは書き換わらない（labs/02）ので、`10.0.0.10` のまま VMware NAT へ出ていく。返事が戻らないのはこのため（NATで解決する。labs/04）
- **`uid` が無い**：`uid` は、どのユーザーのプログラムが送ったか。転送するパケットは外から届いたもので、Router VMのユーザーが送ったものではない
- **`cache iif ens37`**：ens37 から入ってきたパケットとして調べた、という印。転送の経路として調べられている

#### Client VM → Router VM → WAN の経路

```text
Client VM：8.8.8.8 → default via 10.0.0.1     → ens33 から Router VM へ
Router VM：8.8.8.8 → default via 192.168.22.2 → ens33 から VMware NAT へ
```

Client VMもRouter VMも、同じ「ロンゲストマッチで行を選ぶ → `via` の相手に渡す」で、次に渡す相手を決めている。各機器は**次の1台**だけを決め、最終的な届け先までの経路全体は知らない。

### 3. 転送で書き換わるものを観察する（Router VM / Client VM）

Router VMは転送するときに、**MACアドレスとTTLだけ**を書き換えた。IPアドレスはそのままだった。
labs/02 で「見えていないが、起きていること」とした書き換えを、実際に確かめられた。

seq 1 の比較：

| 項目 | ens37（入ってきたとき） | ens33（出ていくとき） | 変化 |
|---|---|---|---|
| MACの送信元 | `00:0c:29:20:6c:ff`（Client VMのens33） | `00:0c:29:73:2d:61`（**Router VMのens33**） | **書き換わった** |
| MACの宛先 | `00:0c:29:73:2d:6b`（Router VMのens37） | `00:50:56:f9:12:3d`（**VMware NAT**） | **書き換わった** |
| `ttl` | `64` | `63` | **1減った** |
| IPの送信元・宛先 | `10.0.0.10 > 8.8.8.8` | `10.0.0.10 > 8.8.8.8` | 変わらない |
| `id` | `18861` | `18861` | 変わらない |
| `tos` / `flags [DF]` / `length 84` | `0x0` / `[DF]` / `84` | 同じ | 変わらない |

- Router VMのens33のMAC（`00:0c:29:73:2d:61`）は labs/01 の `ip link`、VMware NATのMAC（`00:50:56:f9:12:3d`）は labs/01 の `ip neigh` の `192.168.22.2` の行と一致する。Router VMは、ルーティングテーブルで「`192.168.22.2` に渡す」と決め（Result 2）、近隣テーブルでそのMACを調べて宛先にしている
- `id`（IPパケットの通し番号）が両側で同じなので、**同じパケットが転送されている**

| seq | ens37 に届いた時刻 | ens33 から出た時刻 | 差 |
|---|---|---|---|
| 1 | `19:45:54.018043` | `19:45:54.018079` | 約36マイクロ秒 |
| 2 | `19:45:55.031053` | `19:45:55.031062` | 約9マイクロ秒 |
| 3 | `19:45:56.055145` | `19:45:56.055155` | 約10マイクロ秒 |

#### MACアドレスは区間ごとに付け替えられる

```text
LANの区間：Client VMのens33  → Router VMのens37   （MAC：00:0c:29:20:6c:ff → 00:0c:29:73:2d:6b）
WANの区間：Router VMのens33 → VMware NAT         （MAC：00:0c:29:73:2d:61 → 00:50:56:f9:12:3d）
IP      ：10.0.0.10 → 8.8.8.8                    （最初から最後まで同じ）
```

MACアドレスは同じネットワークの中だけで使う宛先なので、ルーターを通るたびに付け替えられる。IPアドレスは最終的な届け先なので、変わらない。

#### TTLはルーターを通るたびに1減る

TTL（Time To Live）は、パケットの**残り寿命**である。ルーターは転送するたびに1減らし、0になったら捨てて、送信元に ICMP の「Time to live exceeded」を返す。経路がループしても、パケットが回り続けないための仕組み。いつ減るのか、初期値の64で足りるのかは [TTL はいつ減り、何のためにあるのか](../../docs/questions/2026-10-25-ttl-decrement-and-purpose.md) を参照。

TTLが変わるとIPヘッダの中身が変わるので、Router VMはIPヘッダのチェックサム（壊れていないかを確かめる値）も計算し直している。`-vv` を付けると `cksum` として表示される。

## Learned

- `ip route get <宛先>` で、その宛先がルーティングテーブルのどの行に当てはまり、どこへ渡されるかを確かめられる
- ルーティングテーブルは、当てはまる行のうちいちばん細かい行を選ぶ（**ロンゲストマッチ**）。default（`0.0.0.0/0`）はいちばん粗い行で、ほかのどの行にも当てはまらないときに使われる
- `via` はネクストホップ。決まるのはMACの宛先で、IPの宛先は変わらない
- 転送するときの経路は `ip route get <宛先> from <送信元> iif <受信したNIC>` で調べる。`iif` を付けないと、自分で送るときの経路になる
- 転送するパケットは、送信元が書き換わらない（`src` が出ない）。Client VMの `10.0.0.10` のまま WAN へ出ていく
- Client VMもRouter VMも、ルーティングテーブルで**次の1台**だけを決める。経路全体を知っている機器はいない
- 転送で書き換わるのは**MACアドレスとTTL**。MACアドレスは区間ごとに付け替えられ、IPアドレスは最初から最後まで変わらない
- TTLはルーターを通るたびに1減り、0になると捨てられる。経路がループしても、パケットが回り続けないため
