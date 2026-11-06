# nftables

## nftables とは

nftables は、**Linux カーネルの中を通るパケットを、どう扱うか（通す・捨てる・書き換える）を設定する仕組み**です。NAT もファイアウォールも、nftables で設定します。

実際にパケットを扱うのは、カーネルの中の **Netfilter** という部分です。nftables は、Netfilter に「こういうパケットは、こう扱って」というルールを渡す道具（`nft` コマンドと、その裏の仕組み）です。

```text
 あなた ──→ nft コマンド ──→ Netfilter（カーネル）──→ パケットを通す・捨てる・書き換える
            └ nftables ┘
```

名前の注意：

- 正しい名前は **nftables**（複数形）。`nftable` とは言わない
- 「NAT テーブル」は nftables のことではない。nftables の中に作る **table の1つ**（下の「table・chain・rule」の節）

このラボでは、Debian 13 に入っている nftables v1.1.3 を使います。

---

## カーネルとの関係

nftables のルールは、カーネルの中で使われます。カーネルとは何か（カーネル空間とユーザー空間、カーネルの動き方を決める3種類のもの）は、[Linux カーネル](README.md) にまとめています。

nftables が変えるのは、3種類のうちの「**参照する表・ルール**」です。カーネルのプログラムには手を加えず、カーネルがパケットごとに見るルールを、渡したり差し替えたりします。

### ルールは「小さなプログラム」として渡される

正確に言うと、`nft` コマンドは、書いたルールを**カーネルが実行できる形**に変換して渡します。カーネルの中の nf_tables は、関所に来たパケットごとにそれを実行します。

```text
 あなた：  oifname "ens33" masquerade
              ↓ nft が変換する
 カーネル：「出ていく NIC の名前を読む → ens33 と比べる → 同じならマスカレード」
           という手順を、関所に来たパケットごとに実行する
```

渡しているのは単なる値ではなく、カーネルの中で動く**手順**です。それでも、カーネルのプログラム自体は変わりません。カーネルが「関所で実行するもの」として受け付けた手順を、差し替えているだけです。

---

## ルーティングテーブルとの違い

どちらもカーネルの中にある表ですが、**決めることが違います**。

| | ルーティングテーブル | nftables のルール |
|---|---|---|
| 決めること | パケットを**どこへ**渡すか | パケットを**どう扱うか**（通す・捨てる・書き換える） |
| 1行の中身 | 宛先の範囲と、渡す相手 | 条件と動作（「ens33 から出るなら、送信元を書き換える」など） |
| 行の選び方 | 当てはまる行のうち、いちばん細かい1行（ロンゲストマッチ） | 上から順に見ていき、条件に合った行の動作を実行する |
| 中身が空のとき | どこにも送れない | 何もしない（全部そのまま通す） |
| 見るコマンド | `ip route` | `sudo nft list ruleset` |

似ているところ：

- どちらも、パケットが通るたびに見られる
- どちらも、コマンドで中身を見たり、足したりできる
- どちらも、コマンドで足しただけでは再起動すると消える

ルーティングテーブルは「**行き先を決める表**」、nftables は「**関所に置くルールの一覧**」です。両者は別の表で、協力して動いています。

---

## 関所（フック）

関所の基本（5つの関所、パケットが通る3つの道、関所ごとに置くもの）は、図と表を使って [関所（Netfilter のフック）](netfilter-hooks.md) にまとめています。この節は、関所の正体と確かめ方を詳しく扱います。

### 関所の正体

関所の正体は、**カーネルの中のパケットを処理するプログラムの、決まった場所にある「呼び出し口」**です。これを **Netfilter のフック**と呼びます。

カーネルのパケット処理の途中には、次のような目印があらかじめ埋め込まれています。

```text
 カーネルのパケット処理（イメージ）

   パケットを受け取る
   ─── ① prerouting ：「ここに登録されているものがあれば、呼び出す」───
   ルーティングテーブルを見て、行き先を決める
   ├─ 自分宛てなら
   │    ─── ② input ：「ここに登録されているものがあれば、呼び出す」───
   │    受け取る
   └─ 転送するなら
        TTL を 1 減らす
        ─── ③ forward ：「ここに登録されているものがあれば、呼び出す」───
        ─── ⑤ postrouting ：「ここに登録されているものがあれば、呼び出す」───
        NIC から送り出す

   自分で送るなら
   ─── ④ output ：「ここに登録されているものがあれば、呼び出す」───
   ルーティングテーブルを見て、行き先を決める
   ─── ⑤ postrouting ───
   NIC から送り出す
```

| 性質 | 説明 |
|---|---|
| 関所の場所はカーネルが決めている | どこに呼び出し口があるかは、カーネルのプログラムに最初から書かれている。自分で増やしたり、場所を変えたりはできない |
| 何も登録されていなければ素通り | `sudo nft list ruleset` が空の Router VM は、どの関所も素通りしている |
| chain を作ると登録される | `hook postrouting` を付けて chain を作ると、⑤ の呼び出し口に登録される。それ以降、⑤ を通るパケットはその chain のルールで調べられる |
| NAT の chain は `nf_nat` を通して登録される | `type nat` の chain は関所に直接登録されない。カーネルの NAT の部品（`nf_nat`）が関所に登録され、そこから自分の chain が呼ばれる。最初の NAT の chain を作ると、`nf_nat` は prerouting・input・output・postrouting の4つの関所すべてに登録される（labs/04 で確認） |

空港にたとえると、関所は「検査場を置ける場所」として、建物を作ったときに最初から決められている場所です。そこに検査場（chain）を開いて初めて、検査が行われます。

### 関所は5つ（IPv4）

```text
                                   ┌──→ ② input ──→ Router VM 自身が受け取る
                                   │
 NIC に届く ──→ ① prerouting ──→ ルーティング判断
                                   │
                                   └──→ ③ forward ──→ ⑤ postrouting ──→ NIC から出ていく
                                                          ↑
 Router VM 自身が送る ──→ ④ output ──→ ルーティング判断 ──┘
```

| 関所 | 名前の意味 | いつ通るか | 主な使い道 |
|---|---|---|---|
| ① prerouting | ルーティング判断の**前** | 届いた直後 | 宛先の書き換え（DNAT、ポート転送） |
| ② input | 自分宛てに**入る** | Router VM 自身宛てのとき | Router VM を守るファイアウォール |
| ③ forward | **転送**する | 転送するとき | LAN と WAN の間のファイアウォール（labs/05） |
| ④ output | 自分が**出す** | Router VM 自身が送るとき | — |
| ⑤ postrouting | ルーティング判断の**後** | 出ていく直前 | **送信元の書き換え（マスカレード）** |

### パケットによって通る関所が違う

パケットは、5つ全部を通るわけではありません。

| パケット | 通る関所 | 例 |
|---|---|---|
| 転送するパケット | ① prerouting → ③ forward → ⑤ postrouting | Client VM から 8.8.8.8 への ping |
| Router VM 自身宛てのパケット | ① prerouting → ② input | Mac から Router VM への SSH |
| Router VM 自身が送るパケット | ④ output → ⑤ postrouting | Router VM の `apt update` |

### 関所の名前を確かめる方法

関所の名前は、chain を作るときに指定します。名前は次の方法で確かめられます。

**1. マニュアル（`man nft`）**

`man nft` を開いて `/ADDRESS FAMILIES` と入力すると、ファミリーごとに使える関所の一覧が出てきます（`n` で次の検索結果へ、`q` で終了）。

| ファミリー | 使える関所 |
|---|---|
| `ip`（IPv4）、`ip6`（IPv6）、`inet`（両方） | `prerouting`、`input`、`forward`、`output`、`postrouting`（`inet` は `ingress` も） |
| `arp` | `input`、`output` |
| `bridge` | `prerouting`、`input`、`forward`、`output`、`postrouting` |
| `netdev` | `ingress`、`egress` |

`CHAINS` の章には、chain の種類（`type`）ごとに置ける関所が書かれています（下の「chain」の節）。

**2. 今のカーネルの関所を見る（`sudo nft list hooks`）**

各関所と、そこに今何が登録されているかを表示します。chain を作る前と後で実行すると、自分の chain が関所に登録されたことを確かめられます。

chain を作る前の Router VM（2026-11-05）：

```text
kajih@debian-router:~$ sudo nft list hooks
family ip {
	hook postrouting {
		-0000000225 apparmor_ip_postroute
	}
}
family ip6 {
	hook postrouting {
		-0000000225 apparmor_ip_postroute
	}
}
```

- `-0000000225` は priority（順番）。小さいほど先に呼ばれる
- `apparmor_ip_postroute` は、AppArmor（Debian に最初から入っているセキュリティの仕組み）のためにカーネルが登録した処理。パケットを書き換えたり捨てたりはしない
- conntrack は出ていない。今の Linux では、conntrack は必要になるまで関所に登録されない

NAT の chain を作り、`masquerade` の rule を入れた後の Router VM（2026-11-05）：

```text
kajih@debian-router:~$ sudo nft list hooks
family ip {
	hook prerouting {
		-0000000400 ipv4_conntrack_defrag [nf_defrag_ipv4]
		-0000000200 ipv4_conntrack_in [nf_conntrack]
		-0000000100 nf_nat_ipv4_pre_routing [nf_nat]
	}
	hook input {
		+0000000100 nf_nat_ipv4_local_in [nf_nat]
		+2147483647 nf_confirm [nf_conntrack]
	}
	hook output {
		-0000000400 ipv4_conntrack_defrag [nf_defrag_ipv4]
		-0000000200 ipv4_conntrack_local [nf_conntrack]
		-0000000100 nf_nat_ipv4_local_fn [nf_nat]
	}
	hook postrouting {
		-0000000225 apparmor_ip_postroute
		+0000000100 nf_nat_ipv4_out [nf_nat]
		+2147483647 nf_confirm [nf_conntrack]
	}
}
```

| 登録された時点 | 現れたもの |
|---|---|
| NAT の chain を作ったとき | `[nf_nat]`（NAT の部品）が prerouting・input・output・postrouting に |
| `masquerade` の rule を入れたとき | `[nf_conntrack]`（conntrack）と `[nf_defrag_ipv4]`（組み立て直し）が現れた。`masquerade` は帰りの書き戻しに conntrack の記録が必要だから |

prerouting の中の並び（priority の小さい順）に、帰りの書き戻しの仕組みがそのまま表れています。

```text
 ① prerouting
   -400  組み立て直す（defrag）
   -200  conntrack で記録を調べる   ← 「これは 10.0.0.10 の通信の返事だ」とわかる
   -100  NAT で宛先を書き戻す        ← その記録を使って 10.0.0.10 に戻す
   ────  ルーティング判断             ← 宛先が 10.0.0.10 なので ens37 へ
```

- `nf_confirm`（priority `+2147483647` = 最大）は、いちばん最後に新しい通信の記録を確定する。途中で捨てられたパケットの記録は残さない
- forward には何も登録されない。転送するパケットも prerouting と postrouting を通るので、そこで conntrack と NAT が済む

### `list ruleset` と `list hooks` の違い

2つのコマンドは、**同じ関所を、違う立場から見ている**。

| コマンド | 見ているもの | たとえ |
|---|---|---|
| `sudo nft list ruleset` | **自分が nftables に入れたもの**の中身（table・chain・rule） | チェックリストのファイルの中身 |
| `sudo nft list hooks` | **カーネルの関所ごとに、何が登録されているか** | 建物の案内図。どの検査場所に、誰の検査場が、どの順で開いているか |

| | `list ruleset` | `list hooks` |
|---|---|---|
| rule の中身（条件・動作） | 出る | 出ない |
| 自分の chain がどの関所に、何番目に登録されたか | 関所の名前と priority は出る | 出る（ほかの登録との順番もわかる） |
| nftables 以外が登録したもの（AppArmor、conntrack など） | 出ない | **出る** |
| 使う場面 | 自分が入れた設定が正しいかを見る（ふだんはこちら） | 関所で実際に何が、どの順で動くかを見る。ほかのツールとぶつかっていないかを調べる |

chain を作る前の Router VM では、`list ruleset` は空なのに、`list hooks` には AppArmor が出た。AppArmor は nftables の外にあるからである。

---

## table・chain・rule

nftables のルールは、**3段の入れ物**に整理して置きます。

```text
table（テーブル）    … chain をまとめる、いちばん外側の入れ物
 └─ chain（チェイン） … rule を並べた一覧。関所に置く
     └─ rule（ルール） … 条件と動作の1行
```

### たとえ：空港の検査

| nftables | 空港 | 例 |
|---|---|---|
| 関所（フック） | 検査をする**場所**（建物に最初から決まっている） | 出国審査、手荷物検査 |
| table | 担当部署ごとの**ファイル** | 保安検査の部署のファイル、税関のファイル |
| chain | ある場所で使う**チェックリスト** | 手荷物検査場のチェックリスト |
| rule | チェックリストの**1項目** | 「液体を持っていたら没収」 |

関所はカーネルが用意しているもの、table・chain・rule は自分で作るものです。

### rule：条件と動作の1行

**rule は、nftables でいちばん小さい単位**です。「**こういうパケットなら（条件）、こうする（動作）**」という1行です。

```text
oifname "ens33"   masquerade
└── 条件 ──┘      └─ 動作 ─┘
出ていく NIC が     送信元を、出ていく NIC の
ens33 なら          IP に書き換える
```

| 条件の例 | 意味 |
|---|---|
| `oifname "ens33"` | 出ていく NIC が ens33 |
| `iifname "ens37"` | 入ってきた NIC が ens37 |
| `ip saddr 10.0.0.0/24` | 送信元 IP が `10.0.0.0/24` の中 |
| `tcp dport 22` | 宛先ポートが 22（SSH） |

| 動作の例 | 意味 |
|---|---|
| `masquerade` | 送信元を、出ていく NIC の IP に書き換える |
| `accept` | 通す |
| `drop` | 黙って捨てる |

- 条件は省略できる。省略すると、すべてのパケットに当てはまる
- **何のためか**：「どのパケットを、どう扱うか」を1つずつ書くため。nftables で実際に何かをするのは、rule だけ

### chain：rule を並べた一覧で、どの関所に置くかを決める

**chain は、rule を上から順に並べたリスト**です。関所に置くのは、table ではなく **chain** です。

chain を作るときに、次のことを決めます。

| 決めること | 書き方 | 今回の値 | 意味 |
|---|---|---|---|
| 何用の chain か | `type` | `nat` | NAT 用 |
| どの関所に置くか | `hook` | `postrouting` | 出ていく直前に、この chain の rule を見る |
| 同じ関所での順番 | `priority` | `srcnat` | 同じ関所にほかの chain があるときの順番。送信元 NAT の標準の値 |
| どれにも当てはまらないとき | `policy` | `accept` | そのまま通す |

chain の種類（`type`）によって、置ける関所が決まっています。

| type | 用途 | 置ける関所（`ip` の場合） |
|---|---|---|
| `filter` | 通す・捨てる（ファイアウォール） | 5つすべて |
| `nat` | IP の書き換え | `prerouting`、`input`、`output`、`postrouting`（**`forward` には置けない**） |
| `route` | 自分が送るパケットの行き先を変え直す | `output` |

`nat` を `forward` に置けないのは、NAT は「ルーティング判断の前（宛先の書き換え）」か「後（送信元の書き換え）」で行うものだからです。

パケットが関所に来ると、その関所に置かれた chain の rule を**上から順に**見ていきます。

```text
 パケットが postrouting の関所に来る
        ↓
 ┌─ chain：postrouting ─────────────────────────────────┐
 │ rule 1：oifname "ens33" masquerade  ← 合えば書き換える  │
 │ rule 2：…                                             │
 │ （どれにも合わなければ policy accept → そのまま通す）     │
 └───────────────────────────────────────────────────────┘
        ↓
 ens33 から出ていく
```

- **何のためか**：rule だけでは「**いつ、どこで**その rule を見るのか」が決まらない。同じ「ens33 なら」という条件でも、置く関所によって意味が変わる。chain は、rule の一覧に置き場所と順番を与える
- 関所に置かない chain も作れる。ほかの chain から「この chain の rule も見て」と呼び出して使うもので、rule が増えて整理したくなったときに使う

### table：chain をまとめる、いちばん外側の入れ物

**table は chain をまとめる入れ物**で、関所には置きません。

| 決めること | 今回の値 | 意味 |
|---|---|---|
| ファミリー（どの種類のパケットを扱うか） | `ip` | IPv4。ほかに `ip6`（IPv6）、`inet`（IPv4 と IPv6 の両方）などがある |
| 名前 | `nat` | 自由に付けられる。習慣で `nat` |

1つの table の中に、別々の関所に置いた chain を入れることもできます。

```text
table ip nat
 ├─ chain postrouting  →  postrouting の関所に置く（送信元の書き換え）
 └─ chain prerouting   →  prerouting の関所に置く（宛先の書き換え。ポート転送など）
```

#### table はパケットの処理に関係しない

カーネルが関所でパケットを調べるとき、見ているのは「その関所に登録された chain と、その priority の順番」だけです。どの table に入っているかは、処理の順番にも結果にも関係しません。

```text
 ⑤ postrouting の関所
   ├─ chain A（priority 100）   ← table X に入っている
   └─ chain B（priority 200）   ← table Y に入っている
   カーネルは priority の順に chain を呼ぶだけ。table の違いは見ていない
```

「rule を chain に入れて関所に置けば良いのでは」という考えは、**処理の仕組みとしては正しい**です。table は、処理のためではなく**管理のための入れ物**です。rule が1つしかないうちは無くても困らないように見えますが、rule が増えたときや、ほかのソフトと同じカーネルを使うときに役に立ちます。

#### table がある理由

**1. 名前がぶつからない**

chain の名前は、**table の中でだけ**重ならなければよい決まりです。

```text
table ip nat        （自分）     → chain postrouting
table ip docker_nat （Docker）   → chain postrouting   ← 同じ名前でもぶつからない
```

table が無いと、カーネル全体で chain の名前が1つの空間を共有することになり、ほかのソフトが同じ名前の chain を作った瞬間にぶつかります。

**2. まとめて扱える（持ち主ごとに分けられる）**

| 操作 | 効果 |
|---|---|
| `sudo nft delete table ip nat` | 自分の NAT の設定だけを丸ごと消す。ファイアウォールの設定やほかのソフトの table には影響しない |
| `sudo nft list table ip nat` | 自分の NAT の設定だけを見る |

Docker などのソフトは、自分専用の table を作って、その中だけを管理します。「自分の table の外には手を出さない」というルールで、複数のソフトが同じカーネルを安全に使えます。

**3. 中の chain どうしで、データを共有できる**

table の中には、chain のほかに **set**（IP アドレスなどの一覧）も置けます。set は同じ table の中の chain から共通で使えます。

```text
table inet fw {
	set admin_ips {                         ← 管理用の IP の一覧
		type ipv4_addr
		elements = { 10.0.0.10 }
	}
	chain input {
		…  ip saddr @admin_ips accept       ← ここでも使う
	}
	chain forward {
		…  ip saddr @admin_ips accept       ← ここでも使う
	}
}
```

一覧を1か所で直せば、両方の chain に効きます。set は table の外からは使えないので、table が「一緒に使うものをまとめる単位」になっています（labs/05 のファイアウォールで使う）。

**4. ファミリーを決める**

table を作るときにファミリー（`ip`、`ip6`、`inet` など）を決めると、中の chain はすべてそのファミリーになります。table が無ければ、chain を作るたびにファミリーを書くことになります。

#### 名前がぶつかってはいけない範囲

table は「**ファミリー + 名前**」の組み合わせで区別されます。ぶつかってはいけないのは、**同じファミリーの中だけ**です。

| 組み合わせ | 一緒に作れるか |
|---|---|
| `table ip nat` と `table ip nat` | **作れない**（同じものとして扱われる） |
| `table ip nat` と `table ip6 nat` | 作れる（ファミリーが違う） |
| `table ip nat` と `table inet nat` | 作れる（ファミリーが違う） |
| `table ip nat` と `table ip filter` | 作れる（名前が違う） |

chain も同じ考え方で、**table の中で**名前が重ならなければよい。入れ子の決まりになっています。

```text
ファミリー ip
 ├─ table nat        ← ファミリーの中で、table の名前は1つずつ
 │   ├─ chain postrouting   ← table の中で、chain の名前は1つずつ
 │   └─ chain prerouting
 └─ table filter
     └─ chain postrouting   ← 別の table なので、同じ名前でもよい
```

**同じ名前で作ろうとすると**：

| コマンド | すでに `table ip nat` があるとき |
|---|---|
| `sudo nft add table ip nat` | **エラーにならず、何もしない**（すでにあるものをそのまま使う） |
| `sudo nft create table ip nat` | エラーになる（`File exists`） |

`add` は「無ければ作る、あればそのまま」という動きです。そのため、2つのソフトがたまたま同じ名前の table を使うと、エラーにならずに**1つの table を共有してしまいます**。

```text
 自分：     sudo nft add table ip nat → chain postrouting を作る
 ソフト X： sudo nft add table ip nat → 同じ table に自分の chain を作る

 ソフト X が終了時に「自分の table を片付ける」：
            sudo nft delete table ip nat
            → 自分の chain まで丸ごと消える
```

table は持ち主ごとに分ける入れ物なので、名前がぶつかると持ち主の区別ができなくなります。そのため、nftables を使うソフトは、ほかとぶつかりにくい自分専用の名前を付けるのが普通です（例：firewalld は `firewalld` という名前の table を使う）。

**このラボでは**：

- Router VM には、nftables を使うほかのソフトを入れていない（`list hooks` で確認済み）。そのため、`nat` のような一般的な名前でも問題ない
- Phase 6 で `routerd` が nftables を操作するようになったら、`routerd` 専用の名前（`routerd_nat` など）にする。手で入れた設定やほかのソフトとぶつからないようにするため

#### table は「ファミリーごと」ではなく「用途・持ち主ごと」

1つの table は必ず1つのファミリーを持ちますが、**同じファミリーの table を、いくつでも作れます**。

```text
table ip nat         ← IPv4、NAT 用（自分）
table ip filter      ← IPv4、ファイアウォール用（自分）
table ip docker_nat  ← IPv4、Docker 用
```

この3つは、どれも同じ `ip` です。それでも table を分けているのは、ファミリーではなく**用途と持ち主**が違うからです。

| 理解のしかた | |
|---|---|
| 「table で、ファミリーごとに chain と rule を管理する」 | △ table を分ける主な理由ではない |
| 「table で、用途・持ち主ごとに chain と rule をまとめて管理する。その table が扱うファミリーも、table で決める」 | ○ |

- **中心になる理由**：1（名前がぶつからない）、2（まとめて扱える）、3（データを共有できる）。どれも「用途・持ち主ごとにまとめる」ためのもの
- **ついでに決めていること**：4（ファミリー）。table の中の chain が、どの種類のパケットを扱うか

空港のたとえで言うと、table は「担当部署のファイル」です。部署（用途・持ち主）ごとにファイルを分けていて、ファイルの表紙に「国際線用」「国内線用」（ファミリー）と書いてある、というイメージです。

| 観点 | table は必要か |
|---|---|
| パケットの処理（どの順に、どう扱うか） | **関係ない**。chain と priority だけで決まる |
| 管理（名前の区切り、持ち主ごとの分離、データの共有、ファミリー） | **必要** |

「NAT テーブル」という呼び方は、前の世代の仕組みである **iptables** から来ています。iptables には `filter`・`nat` などの決まったテーブルが最初から用意されていました。nftables では table を自分で作り、名前も自由に付けます。

### まとめ

| 入れ物 | 何か | 関所との関係 | 何のためか |
|---|---|---|---|
| table | chain をまとめる、いちばん外側の入れ物 | 関係ない（パケットの処理にも関係しない） | 用途・持ち主ごとにまとめる。名前がぶつからない。まとめて消せる。set を共有できる。ファミリーを決める |
| chain | rule を並べた一覧 | **関所に置く** | どの関所で、どの順番で rule を見るかを決める |
| rule | 条件と動作の1行 | chain の中に並べる | どのパケットを、どう扱うかを書く |

---

## 例：マスカレードを入れる（labs/04）

外側から順に、table → chain → rule を作ります。

| 順番 | コマンド | 作るもの |
|---|---|---|
| 1 | `sudo nft add table ip nat` | IPv4 用の table「nat」 |
| 2 | `sudo nft add chain ip nat postrouting '{ type nat hook postrouting priority srcnat; policy accept; }'` | その中に、postrouting の関所に置く NAT 用の chain |
| 3 | `sudo nft add rule ip nat postrouting oifname "ens33" masquerade` | その chain に、「ens33 から出るならマスカレード」の rule |

`'...'` で囲むのは、`{ }` や `;` をシェルに解釈させず、そのまま `nft` に渡すためです。

3つとも入れた後に `sudo nft list ruleset` を実行すると、**入れ子の形**で表示されるはずです。

```text
table ip nat {                                                      ← table
        chain postrouting {                                         ← chain
                type nat hook postrouting priority srcnat; policy accept;
                oifname "ens33" masquerade                          ← rule
        }
}
```

この形は、永続化で書く `/etc/nftables.conf` の書き方と同じです。

### `oifname "ens33" masquerade` の意味

「**ens33 から出ていくパケットなら、送信元を ens33 の IP に書き換える**」という意味です。前半が条件、後半が動作です。

```text
oifname "ens33"   masquerade
└─── 条件 ───┘    └─ 動作 ─┘
```

#### 前半：`oifname "ens33"`（条件）

| 部分 | 意味 |
|---|---|
| `oifname` | **o**utput **i**nter**f**ace **name**。パケットが**出ていく NIC の名前** |
| `"ens33"` | 比べる名前。WAN 側の NIC |

- この rule は postrouting の chain に入っている。postrouting はルーティング判断の**後**なので、この時点では「どの NIC から出ていくか」がもう決まっている。その NIC の名前が `ens33` かどうかを調べる
- 似た条件に `iifname`（**i**nput **i**nterface **name**、入ってきた NIC の名前）がある

**なぜこの条件が必要か**：条件が無いと、postrouting を通る**すべての**パケットが書き換えられます。たとえば、外から Client VM へ転送するパケット（ens37 から出ていく）の送信元まで `10.0.0.1` に書き換わり、Client VM から見て本当の送り主がわからなくなります。「WAN 側へ出ていくときだけ書き換える」ために、出ていく NIC で絞っています。

| 出ていく NIC | 条件に合うか | どうなるか |
|---|---|---|
| ens33（WAN 側） | 合う | 送信元を `192.168.22.129` に書き換える |
| ens37（LAN 側） | 合わない | `policy accept` でそのまま通す |

#### 後半：`masquerade`（動作）

**送信元 IP を、出ていく NIC の IP に書き換える**動作です。ens33 から出ていくので、ens33 の `192.168.22.129` に書き換わります。

似た動作に `snat` があります。

| 書き方 | 書き換える先 | 向いている場面 |
|---|---|---|
| `masquerade` | 出ていく NIC の IP を、**その都度自動で**調べて使う | WAN 側の IP が DHCP でもらったもので、変わることがあるとき |
| `snat to 192.168.22.129` | **自分で指定した** IP に固定 | WAN 側の IP が固定のとき |

Router VM の ens33 の IP は、VMware の DHCP からもらったものです（`ip route` の `proto dhcp`）。変わる可能性があるので、IP を書かなくて済む `masquerade` を使います。家庭用ルーターでも、プロバイダからもらう IP が変わることがあるので、`masquerade` がよく使われます。

#### まとめ

| 部分 | 種類 | 意味 |
|---|---|---|
| `oifname "ens33"` | 条件 | 出ていく NIC が ens33（WAN 側）なら |
| `masquerade` | 動作 | 送信元を、出ていく NIC の IP（`192.168.22.129`）に書き換える |

### 自分で書くのは行きの rule だけ

帰りの返事の宛先を `10.0.0.10` に書き戻す rule は、書く必要がありません。**conntrack もカーネルの関所に登録されて動いていて**、① prerouting の場所で自動で書き戻すからです。

```text
【行き（最初のパケット）】
 ens37 ──→ ルーティング判断 ──→ ⑤ postrouting ──→ ens33
                                    │
                             自分の rule に当てはまる
                             → src を 192.168.22.129 に書き換え
                             → conntrack に記録 ─────────┐
                                                          │
【帰り】                                                   │
 ens37 ←── ③ forward ←── ルーティング判断 ←── ① prerouting ←── ens33
                              │                   │
                    宛先 10.0.0.10 なので      conntrack の記録を見て ←┘
                    ens37 の行が選ばれる       dst を 10.0.0.10 に書き戻す
```

- 自分の NAT の rule が見られるのは、**通信の最初のパケットだけ**。2つ目以降と帰りは、conntrack の記録どおりに書き換える
- 帰りの書き戻しをルーティング判断の**前**に行うのは、先に宛先を戻さないと、Router VM 自身宛てと判断されてしまうから

詳しくは [NAT](../nat.md) を参照してください。

---

## コマンドで入れた設定と、永続化

`nft` コマンドで入れたルールは、カーネルの中にだけあります。再起動すると消えます。

永続化するには、`/etc/nftables.conf` に書き、`nftables.service` を有効にします。

| | 中身 |
|---|---|
| `/etc/nftables.conf` | 起動時に読み込むルール。`sudo nft list ruleset` と同じ入れ子の形で書く |
| `nftables.service` | 起動時に `/etc/nftables.conf` を読み込んでカーネルに渡すサービス。渡したら仕事は終わりで、常に動き続けるプログラムではない |

2026-11-05 時点の Router VM の状態：

```text
kajih@debian-router:~$ sudo nft list ruleset
（何も出ない）
kajih@debian-router:~$ systemctl status nftables
○ nftables.service - nftables
     Loaded: loaded (/usr/lib/systemd/system/nftables.service; disabled; ...)
     Active: inactive (dead)
```

- `disabled`：起動時に自動では動かない
- `inactive (dead)`：今も動いていない（一度も読み込まれていない）

Debian は、nftables をインストールしただけではルールを読み込みません。

### `nft` は `sudo` を付けて実行する

`nft` は `/usr/sbin/nft` にあります。`/usr/sbin` は一般ユーザーの PATH に入っていないので、`sudo` を付けないと `command not found` になります。

---

## ほかのツールとの関係

nftables のほかにも、カーネルにパケットの扱い方のルールを渡すツールがあります。

### カーネルの仕組み（裏側）は2つ

| 仕組み | 説明 |
|---|---|
| **nf_tables** | 今の仕組み。nftables が使う |
| **x_tables** | 前の世代の仕組み。昔の iptables が使う |

どちらも、同じ Netfilter の関所（フック）に登録して動きます。

### 設定するツール（表側）はたくさんある

| ツール | 説明 | 裏側 |
|---|---|---|
| **nftables（`nft`）** | このラボで使うもの | nf_tables |
| **iptables** | 前の世代の定番。今でも解説記事が多い | Debian の今の `iptables` コマンドは、**中身が nf_tables**（`iptables-nft`）。古い x_tables 版（`iptables-legacy`）も選べる |
| ufw、firewalld | ファイアウォールを簡単に設定するためのツール。自分で rule を書く代わりに、nftables や iptables の rule を作ってくれる | nf_tables か x_tables |
| Docker、libvirt、Tailscale など | 自分の通信のために、**勝手に** NAT やファイアウォールの rule を入れるソフト | nf_tables か x_tables |

### 2つのツールで設定すると、両方が同時に効く

片方が無効になるわけではありません。どちらのツールも、最後はカーネルの同じ関所に chain を登録するからです。

```text
 nftables で作った chain ──┐
                           ├──→ ⑤ postrouting の関所に、両方登録される
 iptables で作った chain ──┘
                                パケットは、priority の順に「両方」の chain を通る
```

ぶつかったときの結果は、次のように決まります。

**① 「通す」と「捨てる」がぶつかったら、捨てる方が勝つ**

| 動作 | 効果の範囲 |
|---|---|
| `accept`（通す） | **その chain での判定が終わるだけ**。同じ関所にほかの chain があれば、そちらでも調べられる |
| `drop`（捨てる） | **その場で捨てる**。ほかの chain がどう判定しても、もう戻らない |

```text
 パケット ──→ [ツール A の chain] accept ──→ [ツール B の chain] drop ──→ ✕ 捨てられる
              「通してよい」                 「捨てる」
```

「自分は `accept` と書いたのに通らない」ときは、ほかのツールの chain が `drop` している可能性があります。

**② NAT は、先に書き換えた方が勝つ**

同じ通信に対して両方の chain に NAT の rule があると、priority の順で**先に当てはまった方の書き換え**が conntrack に記録され、その通信ではそれが使われ続けます。後の chain の NAT の rule は使われません。

**③ 設定を丸ごと消してしまうことがある**

Debian の `/etc/nftables.conf` は、標準では1行目が `flush ruleset`（nftables の rule を全部消してから読み込む）です。そのため、`nftables.service` を起動すると、Docker などが入れた table まで消えることがあります。逆に、ほかのツールが起動時に自分の設定を入れ直して、こちらの設定とぶつかることもあります。

**④ 見えない設定ができてしまう**

`iptables-legacy`（x_tables）で入れた rule は、**`sudo nft list ruleset` に出てこない**のに、関所では効いています。「nftables には何も無いのに、パケットが捨てられる」という、原因がわかりにくい状態になります。

### このラボの方針

**nftables だけを使い、ほかのツールは入れない。**

ほかのツールの rule が入っていないかは、次のコマンドで確かめられます。

```bash
sudo iptables --version
sudo iptables -L -n
```

- `--version` の結果が `(nf_tables)` なら、`iptables` コマンドの中身は nftables。`(legacy)` なら、古い x_tables 版
- `-L -n` の結果が空（`policy ACCEPT` の chain が並ぶだけ）なら、iptables の rule は入っていない

---

## このリポジトリでの位置づけ

nftables も Netfilter も、Linux に最初からある仕組みで、このリポジトリでは作りません。作るのは、それを操作する側です（[Router Boundary](../../architecture/router-boundary.md)）。

| Phase | 誰が nftables を操作するか |
|---|---|
| Phase 3〜4 | 人（`nft` コマンド） |
| Phase 5 | Ansible（`configs/nftables/` の設定ファイルを `/etc/nftables.conf` に配る） |
| Phase 6 | `routerd`（NAT Management、Firewall Management） |

---

## 関連

- [nft コマンド](../../tools/nft.md)：コマンドの形式（作る・見る・消す・ファイルから読み込む）
- [NAT](../nat.md)：NAT の本質、行きの書き換えと帰りの書き戻し
- [conntrack](conntrack.md)：通信の記録の表、NAT との関係
- [Routing](../routing.md)：デフォルトゲートウェイ
- [一時的な設定と永続的な設定](../persistent-config.md)
- [nftables はルーティングテーブルのようなものか](../../questions/2026-11-05-nftables-vs-routing-table.md)：この内容のもとになった疑問と回答
- [labs/04-nat](../../../labs/04-nat/README.md)
