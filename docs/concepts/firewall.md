# ファイアウォール

ファイアウォールは、**パケットを通すか捨てるかを、rule で決める仕組み**です。Linux では nftables の rule を関所（input・forward）に置いて実現します（[関所（Netfilter のフック）](kernel/netfilter-hooks.md)、[ADR 008](../decisions/008-nftables-hook-placement.md)）。

このドキュメントでは、次の2つをまとめます。

1. **rule の評価のしかた**：chain の中の rule がどの順番で調べられ、何で結果が決まるか
2. **デフォルト拒否の考え方**：「許可していないものはすべて止める」とはどういうことか

Router VM で実際に rule を書く手順は [labs/05-firewall](../../labs/05-firewall/README.md) で扱います。

---

## 1. rule の評価のしかた

### chain は「上から順に読むチェックリスト」

chain の中には rule が並んでいます。パケットが関所に来ると、**上から順に**1行ずつ調べられます。

ただし、パケットは関所に来る前に、カーネルの処理を通って**下調べ**が済んでいます。rule はその結果を使って条件を調べます。カーネルの処理の流れ全体は [パケットの処理の流れ](kernel/packet-path.md) にあります。

```text
 NIC にパケットが届く
     │
     ▼
 ┌─ カーネルの下調べ（関所に来る前）
 │  NIC ドライバー    ：どの NIC に届いたかを記録する                  → iifname で使う
 │  Ethernet の処理   ：中身が IPv4 か IPv6 かを判断する
 │  IP の処理         ：IP ヘッダーが壊れていないか確かめる（壊れていれば捨てる）
 │                     TCP・UDP・ICMP のどれか、そのヘッダーの位置を記録する
 │  conntrack         ：返事か新しい接続かを決める（① prerouting）     → ct state で使う
 │  ルーティング判断   ：自分宛てか転送かと、出ていく NIC を決める       → oifname で使う（forward のみ）
 └─
     │
     ▼
 パケットが関所（input・forward）の chain に来た
     │
     ▼
 ┌─ rule 1 ── 条件に当たる？ ── はい ──→ 動作を実行（accept なら通す・drop なら捨てる）→ 終わり
 │     │
 │   いいえ
 │     ▼
 ├─ rule 2 ── 条件に当たる？ ── はい ──→ 動作を実行 → 終わり
 │     │
 │   いいえ
 │     ▼
 ├─ rule 3 ── …
 │     │
 │   いいえ（最後まで、どの rule にも当たらなかった）
 │     ▼
 └─ policy（chain の決まり）── accept なら通す・drop なら捨てる
```

| 決まり | 説明 |
|---|---|
| 上から順に調べる | rule を書いた順（`nft list ruleset` で表示される順）に調べる |
| 最初に当たった rule で決まる | `accept` や `drop` の rule に当たったら、そこで終わり。下の rule は見ない |
| 当たらなければ次の行へ | 条件に合わない rule は飛ばして、次の rule を調べる |
| 最後まで当たらなければ policy | chain に書いた `policy` で決まる |

### 下調べと、rule が読む欄の分担

カーネルの下調べは、基本的なことだけです。ポート番号のような細かい欄は、**各 rule が、自分に必要な欄だけを、調べるときにそのつど**パケットから読みます。

| | カーネルの下調べ（関所に来る前） | rule（関所で調べるとき） |
|---|---|---|
| いつ | パケットが届いたときに1回 | rule 1 つごとに |
| 何を調べるか | 届いた NIC、IPv4 か IPv6 か、ヘッダーが正しい形か、プロトコルとヘッダーの位置、`ct state`、出ていく NIC | その rule の条件に必要な欄だけ（`tcp dport 22` なら、プロトコルと宛先ポートの欄） |

たとえば `tcp dport 22 accept` は、カーネルの中では次の手順になります（`sudo nft --debug=netlink add rule ...` で見られます）。

```text
 [ meta load l4proto => reg 1 ]                       ← プロトコルの番号を読む
 [ cmp eq reg 1 0x00000006 ]                          ← 6（TCP）でなければ当たらない → 次の rule へ
 [ payload load 2b @ transport header + 2 => reg 1 ]  ← TCP ヘッダーの 2 バイト目から 2 バイト（宛先ポートの欄）を読む
 [ cmp eq reg 1 0x00001600 ]                          ← 22 でなければ当たらない → 次の rule へ
 [ immediate reg 0 accept ]                           ← ここまで来たら accept
```

「TCP ヘッダーがどこから始まるか」（`transport header`）は、下調べで記録された位置を使っています。

カーネルが見ているのは、ヘッダーの欄の**値**だけです。「SSH のパケット」は、カーネルにとっては「TCP で、宛先ポートの欄が 22」のパケットです。中身が SSH でなくても、宛先ポートが 22 なら `tcp dport 22` の rule に当たります。

### rule の形：条件と動作

1つの rule は「**条件**」と「**動作**」でできています。

```text
 iifname "ens37" oifname "ens33"     accept
 └────────── 条件 ──────────┘     └ 動作 ┘
 「ens37 から入って ens33 から出るなら」  「通す」
```

| よく使う条件 | 意味 |
|---|---|
| `iifname "ens37"` | 入ってきた NIC が ens37 |
| `oifname "ens33"` | 出ていく NIC が ens33 |
| `ip saddr 10.0.0.0/24` | 送信元の IP が 10.0.0.0/24 の範囲 |
| `tcp dport 22` | TCP の宛先ポートが 22（SSH） |
| `ct state established,related` | conntrack の判定が「返事」か「関係するパケット」（[conntrack.md](kernel/conntrack.md)） |
| `ip protocol icmp` | ICMP（ping など） |

条件を複数書くと、**すべてに当たったとき**だけ動作が実行されます（「かつ」の意味）。条件を1つも書かなければ、すべてのパケットが当たります。

### 動作には「そこで終わる」ものと「続ける」ものがある

| 動作 | 種類 | 何をするか |
|---|---|---|
| `accept` | **終わる** | 通す。この chain の残りの rule は見ない |
| `drop` | **終わる** | 黙って捨てる。送り主には何も返さない |
| `reject` | **終わる** | 捨てて、「拒否した」という返事を送り主に送る |
| `counter` | 続ける | 当たったパケットの数を数えて、次の rule へ進む |
| `log` | 続ける | ログに記録して、次の rule へ進む |

`counter` と `log` は、パケットの行き先を決めません。そのため、`counter accept` のように「数えてから通す」と組み合わせて使います。

### 順番が結果を変える

同じ rule でも、並べる順番で結果が変わります。

```text
 例A：SSH を許可したい                     例B：順番を逆にすると
 chain input {                              chain input {
     tcp dport 22 accept      ← 先            ip saddr 192.168.22.0/24 drop   ← 先
     ip saddr 192.168.22.0/24 drop             tcp dport 22 accept
 }                                          }
```

| パケット | 例A | 例B |
|---|---|---|
| 192.168.22.1 から Router VM への SSH | 1行目に当たる → **通る** | 1行目に当たる → **捨てる**（2行目は見ない） |
| 192.168.22.1 から Router VM への ping | 2行目に当たる → 捨てる | 1行目に当たる → 捨てる |

例B では、SSH を許可する rule があるのに、SSH が通りません。**細かい（例外の）rule を上に、大ざっぱな rule を下に**書くのが基本です。

### policy：どの rule にも当たらなかったとき

`policy` は、関所に置いた chain（base chain）に付ける「最後の決まり」です。

```text
chain forward {
	type filter hook forward priority filter; policy drop;
	                                          └── ここ
}
```

| policy | どの rule にも当たらなかったパケット | 書かなかったとき |
|---|---|---|
| `accept` | 通す | **こちらになる** |
| `drop` | 捨てる | — |

- policy に書けるのは `accept` と `drop` だけ（`reject` は書けない）
- 今の `/etc/nftables.conf` の `table inet filter` は policy を書いていないので `accept`。rule も無いので、すべて通っている
- **rule が1つも無い chain に `policy drop` を付けると、その関所を通るパケットはすべて捨てられる**。input でこれをやると、SSH も切れる

### drop と reject の違い

どちらもパケットを捨てますが、送り主から見た結果が違います。

```text
 drop：黙って捨てる
   送り主 ──── パケット ────→ Router VM（捨てる）
   送り主 ……（何も返ってこない。待ち続けて、時間切れになる）

 reject：捨てて、拒否を知らせる
   送り主 ──── パケット ────→ Router VM（捨てる）
   送り主 ←── 「届けられません」（ICMP）や TCP の RST ── Router VM
```

| | `drop` | `reject` |
|---|---|---|
| 送り主に返すもの | 何も返さない | 拒否の返事（ICMP の「到達できない」、`with tcp reset` を付けると TCP の RST） |
| `ping` の見え方 | 返事が来ず、時間切れになる | `Destination Port Unreachable` などがすぐ出る |
| `ssh` の見え方 | つながらないまま待ち続ける | `Connection refused` がすぐ出る |
| 外から見たとき | 「そこに何かあるのか」がわかりにくい | 「何かがあって、拒否している」とわかる |
| 向いている場所 | **WAN 側**（知らない相手に情報も返事も渡さない） | **LAN 側**（自分たちの機器なので、すぐ失敗がわかるほうが便利） |

家庭用ルーターでは、WAN 側から来たものは `drop` にするのが一般的です。

### 補足：accept は「この chain では通す」

`drop` はその場でパケットを捨てて終わりです。一方 `accept` は「**この chain では通す**」という意味で、同じ関所にほかの chain も置かれていれば、そこでも調べられます。

このラボでは、input・forward に置くファイアウォールの chain は1つずつ（`table inet filter` の中）なので、気にしなくて大丈夫です。Docker などのソフトウェアが自分の chain を追加すると、この違いが効いてきます。

---

## 2. デフォルト拒否の考え方

### 2つの書き方

ファイアウォールの書き方は、大きく2つに分かれます。

| | デフォルト許可 | **デフォルト拒否** |
|---|---|---|
| policy | `accept` | **`drop`** |
| rule に書くもの | **止めたいもの**（拒否リスト） | **通したいもの**（許可リスト） |
| 書いていないもの | 通る | **止まる** |
| たとえ | 「危険人物リストに載っている人だけ入れない」 | 「招待リストに載っている人だけ入れる」 |

```text
 デフォルト許可                         デフォルト拒否
 chain forward {                       chain forward {
     policy accept;                        policy drop;
     （止めたいものを列挙）                   （通したいものを列挙）
     tcp dport 23 drop                     ct state established,related accept
     tcp dport 445 drop                    iifname "ens37" oifname "ens33" accept
     …                                 }
 }
```

### デフォルト拒否が選ばれる理由

| 観点 | デフォルト許可 | デフォルト拒否 |
|---|---|---|
| 知らない攻撃 | 拒否リストに無ければ**通ってしまう** | 許可リストに無ければ止まる |
| うっかり公開 | Client VM で新しいサーバーを動かすと、外から届いてしまう | 許可しない限り、外からは届かない |
| rule の数 | 止めたいものは無数にあり、書き切れない | 通したいものは少なく、数行で済む |
| 困ること | 守りが漏れても気づきにくい | 必要なものを許可し忘れると通信が止まる（ただし、すぐ気づく） |

守りの漏れは気づきにくく、許可の漏れはすぐ気づけます。そのため、ファイアウォールでは**デフォルト拒否**が基本です。家庭用ルーターも、WAN から LAN への新しい接続はデフォルトで拒否しています。

### デフォルト拒否で、必ず最初に許可するもの

デフォルト拒否にすると、**返事も止まります**。返事も1つのパケットとして関所を通るからです（[netfilter-hooks.md](kernel/netfilter-hooks.md#返事も同じように関所を通る)）。そのため、ほぼ必ず先頭に次の rule を置きます。

```text
ct state established,related accept
```

| この rule が無いと | この rule があると |
|---|---|
| Client VM が 8.8.8.8 に ping を打つ → 行きは許可の rule で通る → **返事は WAN → LAN なので policy drop で捨てられる** | 返事は `established` なので通る |

返事が多いので、この rule を先頭に置くと、ほとんどのパケットが1行目で決まり、残りの rule を調べずに済みます。

---

## 3. まとめると：forward のデフォルト拒否

1 と 2 を合わせると、家庭用ルーターの forward は次の形になります。

```text
chain forward {
	type filter hook forward priority filter; policy drop;   # デフォルト拒否

	ct state established,related accept                      # rule 1：返事は通す
	iifname "ens37" oifname "ens33" accept                   # rule 2：LAN → WAN の新しい接続は通す
}
```

| パケット | rule 1 | rule 2 | policy | 結果 |
|---|---|---|---|---|
| Client VM → 8.8.8.8（最初の行き、`new`） | 当たらない | **当たる** | — | 通る |
| 8.8.8.8 → Client VM（返事、`established`） | **当たる** | — | — | 通る |
| WAN の機器 → 10.0.0.10:22（新しい接続、`new`） | 当たらない | 当たらない（ens33 から来た） | **drop** | 捨てる |

rule 2 には「WAN → LAN を止める」とはどこにも書いていません。**許可していないから止まる**のが、デフォルト拒否です。

---

## input を同じようにするときの注意

input も同じ考え方でデフォルト拒否にできます。ただし、このラボでは**管理用の SSH が WAN 側（ens33）から Router VM に入っている**ので、順番を間違えると締め出されます。

| やり方 | 結果 |
|---|---|
| 先に `policy drop` にしてから、SSH を許可する rule を足そうとする | `policy drop` にした瞬間に SSH が切れる。rule を足せない |
| 先に `ct state established,related accept` と SSH の許可を入れ、最後に `policy drop` にする | SSH はつながったまま |

input で許可が要るもの（loopback・ICMP・SSH、のちの DHCP・DNS）と、安全に試す手順は、labs/05 で試してからこのドキュメントに追記します。

---

## 関連

- [関所（Netfilter のフック）](kernel/netfilter-hooks.md)：rule を置く場所と、パケットが通る道
- [ADR 008](../decisions/008-nftables-hook-placement.md)：ファイアウォールを input と forward に置く理由
- [conntrack](kernel/conntrack.md)：`ct state` の判定に使う通信の記録
- [nftables](kernel/nftables.md)：table・chain・rule の入れ物の関係
- [ルーターは「返事」と「外からの新しい接続」をどう区別するのか](../questions/2026-11-06-reply-vs-new-connection.md)
- [labs/05-firewall](../../labs/05-firewall/README.md)
