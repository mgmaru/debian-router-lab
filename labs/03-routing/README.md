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

### 4. TTL=1 で送る（Client VM）

Client VMから、TTLを1にして `8.8.8.8` へ `ping` を打った（2026-10-25）。`-t` でTTLの最初の値を指定できる。

```text
kajih@debian-client:~$ ping -c 1 -t 1 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
From 10.0.0.1 icmp_seq=1 Time to live exceeded

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

### 5. 細かい行を足して、選ばれる行を変える（Client VM）

Client VMに、`8.8.8.8` だけを存在しない相手（`10.0.0.99`）に渡す行を一時的に足し、`8.8.8.8` と `8.8.4.4` の経路を調べた（2026-10-25）。

```text
kajih@debian-client:~$ sudo ip route add 8.8.8.8/32 via 10.0.0.99
kajih@debian-client:~$ ip route
default via 10.0.0.1 dev ens33 onlink
8.8.8.8 via 10.0.0.99 dev ens33
10.0.0.0/24 dev ens33 proto kernel scope link src 10.0.0.10
kajih@debian-client:~$ ip route get 8.8.8.8
8.8.8.8 via 10.0.0.99 dev ens33 src 10.0.0.10 uid 1000
    cache
kajih@debian-client:~$ ip route get 8.8.4.4
8.8.4.4 via 10.0.0.1 dev ens33 src 10.0.0.10 uid 1000
    cache
```

最後に、足した行を消して元に戻した。

```bash
sudo ip route del 8.8.8.8/32
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

### 4. TTL=1 で送る（Client VM）

Router VMは、TTLが尽きるパケットを**捨てて、送信元に ICMP の「Time to live exceeded」を返した**。

```text
Client VM：TTL=1 で 8.8.8.8 へ送る
   ↓
Router VM：転送すると決まった → TTLを確認 → 1以下なので転送できない
   ↓ パケットを捨てる
   ↓ 代わりに、送信元（10.0.0.10）へ ICMP「Time to live exceeded」を送る
Client VM：From 10.0.0.1 ... Time to live exceeded と表示
```

| 部分 | 意味 |
|---|---|
| `From 10.0.0.1` | エラーを返したのは**Router VM**。`8.8.8.8` からの返事ではない |
| `icmp_seq=1` | 1つ目の echo request についてのエラー。エラーの中に捨てたパケットの先頭が入っているので、Client VMはどれについてのエラーか分かる |
| `Time to live exceeded` | ICMP の Time Exceeded（Type 11）、Code 0（転送中にTTLが尽きた） |
| `0 received, +1 errors` | echo reply は0個、エラーが1個届いた |
| `100% packet loss` | `8.8.8.8` からの返事は来なかった |

#### labs/02 で黙って捨てたときとの違い

| 場面 | Router VMの動き | Client VMから見えること |
|---|---|---|
| `ip_forward = 0`（labs/02） | 黙って捨てる | 何も返らない（`100% packet loss` だけ） |
| TTLが尽きた（今回） | 捨てて、**ICMPで知らせる** | `From 10.0.0.1 ... Time to live exceeded` |

このエラーは、NATが無くても Client VM に届いた。返したのはRouter VM自身で、Client VMと同じLAN（`10.0.0.0/24`）にいるので、帰り道に困らない。

#### traceroute の仕組み

`traceroute` は、TTLを1、2、3…と増やしながら送り、各ルーターが返す「Time to live exceeded」の送信元から、経路を1台ずつ調べる。

```text
TTL=1 → 1台目（10.0.0.1）が返す        ← 今回確かめた
TTL=2 → 2台目（192.168.22.2）が返す…はず
```

今はNATが無いので、2台目から先の返事は Client VM に戻らない。NATを設定した後（labs/04）に試すと、経路が1台ずつ見えるはずである。

### 5. 細かい行を足して、選ばれる行を変える（Client VM）

`8.8.8.8/32` の行を足すと、`8.8.8.8` だけが `10.0.0.99` へ渡されるようになった。`8.8.4.4` は、これまで通り default の `10.0.0.1` へ渡される。

| 宛先 | `8.8.8.8/32`（`8.8.8.8` だけ） | `10.0.0.0/24` | `default`（`/0`、すべて） | 選ばれた行 |
|---|---|---|---|---|
| `8.8.8.8` | **当てはまる** | 当てはまらない | 当てはまる | `/32` → `via 10.0.0.99` |
| `8.8.4.4` | 当てはまらない | 当てはまらない | 当てはまる | `/0` → `via 10.0.0.1` |

- `8.8.8.8` は `/32` の行と default の行の**両方に当てはまる**。範囲が狭い（プレフィックス長が長い）`/32` の行が選ばれた。Result 1 で書いた**ロンゲストマッチ**を、行を足して実際に確かめられた
- `/数字` はプレフィックス長で、サブネットの `/24` と同じもの。ルーティングテーブルでは「この行が当てはまる宛先の範囲」を表す。詳しくは [ロンゲストマッチとは何か：`/32` と `/0` の数字はサブネットと同じものか](../../docs/questions/2026-10-25-longest-match-and-prefix-length.md) を参照
- `ip route` の表示では、`/32` は省略されて `8.8.8.8` とだけ出る
- `ip route add` で足した行は一時的なもので、再起動すると消える。今回は `ip route del` で消して元に戻した
- 行を1つ足すだけで、パケットの行き先が変わる。`10.0.0.99` は存在しないので、この状態の `8.8.8.8` への通信は届かない。ルーティングテーブルの設定を間違えると、特定の宛先にだけ届かなくなる

## Learned

- `ip route get <宛先>` で、その宛先がルーティングテーブルのどの行に当てはまり、どこへ渡されるかを確かめられる
- ルーティングテーブルは、当てはまる行のうちいちばん細かい行を選ぶ（**ロンゲストマッチ**）。default（`0.0.0.0/0`）はいちばん粗い行で、ほかのどの行にも当てはまらないときに使われる
- `via` はネクストホップ。決まるのはMACの宛先で、IPの宛先は変わらない
- 転送するときの経路は `ip route get <宛先> from <送信元> iif <受信したNIC>` で調べる。`iif` を付けないと、自分で送るときの経路になる
- 転送するパケットは、送信元が書き換わらない（`src` が出ない）。Client VMの `10.0.0.10` のまま WAN へ出ていく
- Client VMもRouter VMも、ルーティングテーブルで**次の1台**だけを決める。経路全体を知っている機器はいない
- 転送で書き換わるのは**MACアドレスとTTL**。MACアドレスは区間ごとに付け替えられ、IPアドレスは最初から最後まで変わらない
- TTLはルーターを通るたびに1減り、0になると捨てられる。経路がループしても、パケットが回り続けないため
- TTLが尽きるパケットを受け取ったルーターは、捨てて、送信元に ICMP の「**Time to live exceeded**」を返す。`ip_forward = 0` で黙って捨てるのとは違い、送信元に知らせる。`traceroute` はこれを使って経路を調べる
- `ip route add <宛先>/<プレフィックス長> via <ネクストホップ>` で行を足し、`ip route del` で消せる（一時的）。細かい行を足すと、default より優先される
- `/数字`（プレフィックス長）は、サブネットの数字と同じもの。ルーティングテーブルでは、その行が当てはまる宛先の範囲を表す
