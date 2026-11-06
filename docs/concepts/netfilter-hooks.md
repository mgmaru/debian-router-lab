# 関所（Netfilter のフック）

このドキュメントは、NAT とファイアウォールの土台になる「関所」の基本だけをまとめたものです。
`nft list hooks` の読み方や priority などの詳しい話は、[nftables.md の「関所（フック）」](nftables.md#関所フック) にあります。

---

## 一言でいうと

**関所は、Router VM のカーネルの中で、パケットが必ず通る「チェックポイント」です。**

- 関所は全部で **5つ**ある（IPv4 の場合）
- 場所は**カーネルが最初から決めている**。自分で増やしたり、場所を変えたりはできない
- 何も置かなければ、パケットは**素通り**する
- nftables で **chain**（ルールの一覧）を関所に置くと、そこを通るパケットがルールで調べられる

```text
 関所 ＝ 検査をする「場所」（カーネルが用意している）
 chain ＝ その場所に置く「検査のチェックリスト」（自分で作る）
```

**関所は Router VM の中にあるものです**。Client VM や Mac のような機器の名前ではありません。

---

## 5つの関所

| 番号 | 関所の名前 | 名前の意味 | どこにあるか |
|---|---|---|---|
| ① | **prerouting** | ルーティングの「前」 | NIC にパケットが届いた直後。行き先はまだ決まっていない |
| ② | **input** | 「入る」 | Router VM **自身宛て**と決まったパケットが、受け取られる直前 |
| ③ | **forward** | 「転送する」 | **ほかの機器宛て**と決まったパケットが、転送される途中 |
| ④ | **output** | 「出す」 | Router VM **自身が作った**パケットが、出ていく途中 |
| ⑤ | **postrouting** | ルーティングの「後」 | NIC からパケットが出ていく直前 |

---

## 関所の地図

Router VM の中を上から見た図です。左から入って右へ出ていきます。

```text
                                       ┌─→ ② input ─→ Router VM のプログラム（sshd など）が受け取る
                                       │
 NIC に届く ─→ ① prerouting ─→ ルーティング判断
                                       │
                                       └─→ ③ forward ─→ ⑤ postrouting ─→ NIC から出る
                                                              ▲
                                                              │
 Router VM のプログラム ─→ ルーティング判断 ─→ ④ output ──────┘
```

④ output から来たパケットも、③ forward から来たパケットも、最後は同じ ⑤ postrouting を通って NIC から出ていきます。

**ルーティング判断**が分かれ道です。ここで「Router VM 自身宛てか、ほかの機器宛てか」が決まり、通る関所が変わります。

---

## パケットが通る道は3つ

どのパケットも、次の3つの道のどれか1つを通ります。**5つ全部を通るパケットはありません。**

| 道 | どんなパケットか | 通る関所 |
|---|---|---|
| **道1：通り抜ける** | ほかの機器から来て、ほかの機器へ行く | ① prerouting → ③ forward → ⑤ postrouting |
| **道2：Router VM に届く** | Router VM 自身宛て | ① prerouting → ② input |
| **道3：Router VM から出る** | Router VM 自身が作った | ④ output → ⑤ postrouting |

このラボの機器で、それぞれの例を見ていきます。

```text
 Client VM                  Router VM                                  インターネット
 10.0.0.10  ───  ens37（LAN側）10.0.0.1 ／ ens33（WAN側）192.168.22.129  ───  8.8.8.8
```

### 道1：通り抜ける（Client VM → 8.8.8.8 の ping）

```text
 Client VM ─→ ens37 ─→ ① prerouting ─→ ルーティング判断 ─→ ③ forward ─→ ⑤ postrouting ─→ ens33 ─→ 8.8.8.8
                                       「8.8.8.8 は自分じゃない。
                                         ens33 から転送する」
```

Router VM はこのパケットを「受け取る」わけではありません。**中を通り抜けさせるだけ**です。だから ② input は通りません。

### 道2：Router VM に届く（Mac → Router VM への SSH）

```text
 Mac ─→（Windows・VMware を経由）─→ ens33 ─→ ① prerouting ─→ ルーティング判断 ─→ ② input ─→ sshd
                                                         「192.168.22.129 は自分だ。
                                                           受け取る」
```

宛先が Router VM 自身なので、③ forward は通りません。

### 道3：Router VM から出る（Router VM の `apt update`、SSH の返事）

```text
 apt ─→ ルーティング判断 ─→ ④ output ─→ ⑤ postrouting ─→ ens33 ─→ インターネット
```

Router VM のプログラムが作ったパケットは、① prerouting も ③ forward も通りません。

### 返事も、同じように関所を通る

返事のパケットも、普通のパケットと同じように道を選びます。**行きと帰りは、別々のパケットとして関所を通ります。**

| 通信 | 行き | 帰り（返事） |
|---|---|---|
| Client VM → 8.8.8.8 の ping | 道1（① → ③ → ⑤） | 道1（① → ③ → ⑤）。8.8.8.8 から Client VM へ通り抜ける |
| Mac → Router VM の SSH | 道2（① → ②） | 道3（④ → ⑤）。sshd が返事を作って出す |
| Router VM の `apt update` | 道3（④ → ⑤） | 道2（① → ②）。返事は Router VM 宛てに届く |

ここから次のことがわかります。

- ③ forward は「LAN → WAN」と「WAN → LAN」の**両方の向き**のパケットが通る
- ② input には「外から Router VM への新しい接続」と「Router VM が出した通信への返事」の両方が届く

そのため、ファイアウォールのルールでは「どちら向きか」「新しい接続か、返事か」を区別する必要があります（返事かどうかは conntrack の記録でわかります。[conntrack.md](conntrack.md)）。

---

## どの関所に何を置くか

### 関所ごとに、通るパケットとわかっていることが違う

関所そのものに役割が決められているわけではありません。違うのは**場所**です。場所が違うと「どのパケットが通るか」と「その時点で何がわかっているか」が変わり、その結果、関所ごとに向いている仕事が分かれます。

| 関所 | 通るパケット | その時点でわかっていること | だから向いている仕事 |
|---|---|---|---|
| ① prerouting | 届いたパケットすべて | 入ってきた NIC。**出口はまだ決まっていない** | 宛先の書き換え（DNAT）。行き先を決める前に宛先を変えられる |
| ② input | Router VM 宛てだけ | 自分宛てだと決まった | Router VM を守るファイアウォール |
| ③ forward | 通り抜けるものだけ | 入口と出口の両方 | LAN 側の機器を守るファイアウォール |
| ④ output | Router VM が作ったものだけ | 出口 | Router VM が出す通信の制限（あまり使わない） |
| ⑤ postrouting | 出ていくパケットすべて | **出口が決まった** | 送信元の書き換え（マスカレード）。出口の NIC の IP がわかるから |

「向いている仕事」は慣習で、カーネルが強制しているわけではありません。たとえば ① prerouting に filter の chain を置き、届いた直後に捨てることもできます。

ただし、カーネルが制限しているところが1つだけあります。**NAT の chain（`type nat`）は ③ forward には置けません**。置けるのは prerouting・input・output・postrouting です。filter の chain は5つのどこにでも置けます。

### このラボで置くもの

| 関所 | このラボでの例 | 使う lab |
|---|---|---|
| ① prerouting | —（NAT の書き戻しは conntrack が自動で行うので、rule は書かない） | — |
| ② input | **Router VM を守る**ファイアウォール：「SSH は許可、ほかは拒否」 | labs/05 |
| ③ forward | **LAN 側の機器を守る**ファイアウォール：「LAN → WAN は許可、WAN → LAN の新しい接続は拒否」 | labs/05 |
| ④ output | — | — |
| ⑤ postrouting | 送信元の書き換え：`oifname "ens33" masquerade` | labs/04（済み） |

labs/04 では ⑤ postrouting に NAT の chain を置きました。labs/05 では ② input と ③ forward にファイアウォールの chain を置きます。

この置き場所は、このラボの設計方針として [ADR 008](../decisions/008-nftables-hook-placement.md) に決めた理由とあわせて記録しています。

今の `/etc/nftables.conf` にある `table inet filter` には、input・forward・output の3つの chain がすでにあります。ただ、中にルールが1つもないので、すべて素通りしています。

```text
table inet filter {
	chain input {
		type filter hook input priority filter;     ← ② input に置いた chain（中身は空）
	}
	chain forward {
		type filter hook forward priority filter;   ← ③ forward に置いた chain（中身は空）
	}
	chain output {
		type filter hook output priority filter;    ← ④ output に置いた chain（中身は空）
	}
}
```

`hook input` の部分で、どの関所に置くかを指定しています。chain の名前（`input` など）は自分で決めたラベルです。関所と同じ名前にしておくとわかりやすい、というだけです。

---

## ファイアウォールの「守る対象2つ」と関所

| 守る対象 | パケットが通る道 | ルールを置く関所 | ルールを実行する場所 |
|---|---|---|---|
| Router VM 自身 | 道2（Router VM に届く） | ② input | Router VM |
| LAN 側の機器（Client VM など） | 道1（通り抜ける） | ③ forward | **Router VM** |

LAN 側の機器を守るルールも、**Router VM の中**で実行されます。WAN から Client VM へ行くパケットは必ず Router VM を通り抜けるので、Router VM の ③ forward で落とせば Client VM には届きません。Client VM の側では何もしていません。

### ③ forward で見える IP アドレス

NAT をしていても、③ forward では **Client VM の本当のアドレス（10.0.0.10）** が見えます。書き換えが ③ forward より前か後かで決まります。

| パケット | 書き換えが起きる関所 | ③ forward で見えるアドレス |
|---|---|---|
| 行き（Client VM → 8.8.8.8） | ⑤ postrouting で、送信元を 10.0.0.10 → 192.168.22.129 に書き換える（③ より**後**） | 送信元は **10.0.0.10** のまま |
| 帰り（8.8.8.8 → Client VM） | ① prerouting で、宛先を 192.168.22.129 → 10.0.0.10 に書き戻す（③ より**前**） | 宛先は **10.0.0.10** に戻っている |

そのため ③ forward のルールは、NAT のことを気にせずに、LAN のアドレスや NIC の名前（ens37・ens33）で書けます。

---

## よくある勘違い

| 勘違い | 正しくは |
|---|---|
| forward は LAN 側の機器のこと | forward は Router VM の中の関所の名前。LAN 側の機器を守るルールを**置く場所**が forward |
| 関所は NIC ごとにある | 関所はカーネルに1組だけある。ens33 から来たパケットも ens37 から来たパケットも、同じ関所を通る。NIC の区別はルールの条件（`iifname`・`oifname`）で行う |
| パケットは5つの関所を全部通る | 通るのは3つの道のどれか1つ。道1でも3つ、道2と道3は2つだけ |
| 返事のパケットは関所を通らない | 返事も1つのパケットとして、普通に関所を通る。ルールで止めれば返事も止まる |
| 通り抜けるパケットも input を通る | input を通るのは Router VM 自身宛てのパケットだけ。通り抜けるパケットは forward を通る |
| 関所に何も置かないと通信が止まる | 何も置かなければ素通り。止めたいときにルールを置く |
| 出ていく先（出口の NIC）は、ルーティングテーブルと関所で決めている | 出口を決めるのは**ルーティングテーブルだけ**。関所は決まった出口を見て、通す・捨てる・書き換えるを決める（`oifname "ens33"` は「出口が ens33 なら」という条件）。ただし ① prerouting で宛先を書き換えると、新しい宛先でルーティングされるので、出口が**間接的に**変わる（NAT の返事が ens37 から出るのはこのため）。③ forward と ⑤ postrouting では、ルーティング判断が終わっているので出口は変わらない |

---

## 関連

- [nftables](nftables.md)：関所の正体、`nft list hooks` での確認方法、table・chain・rule
- [conntrack](conntrack.md)：返事かどうかを見分けるための通信の記録
- [NAT](nat.md)：⑤ postrouting で送信元を書き換える仕組み
- [labs/05-firewall](../../labs/05-firewall/README.md)：② input と ③ forward にルールを置く実験
