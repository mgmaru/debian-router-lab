# ルーターは「返事」と「外からの新しい接続」をどう区別するのか：forward は関所か、LAN → WAN と WAN → LAN の見分け方、conntrack の判定と ct state

調査日：2026-11-06

関連：

- [関所（Netfilter のフック）](../concepts/netfilter-hooks.md)（5つの関所と、パケットが通る3つの道）
- [conntrack](../concepts/conntrack.md)（通信の記録の表、1行が前半と後半の2つの向きでできていること）
- [nftables](../concepts/nftables.md)
- [labs/05-firewall](../../labs/05-firewall/README.md)
- [WANの入口は攻撃者の標的になるのか](2026-10-08-is-wan-entrance-a-target.md)

## 疑問

> **質問70**：Firewall の守る対象は2つあるとのことだが、LAN 側の機器を forward というのか？ forward はルーターの関所か？

> **質問71**：ルーターは、LAN から WAN へのパケットと WAN から LAN へのパケットを、どのように区別しているのか？

> **質問72（追加質問）**：本当に聞きたかったのは、「Client VM が始めた通信への返事（通したい）」と「外から始まった新しい接続（止めたい）」をどう分けるのか、ということ。この2種類を区別するのが大事だと思っている。

## 結論

| 質問 | 答え |
|---|---|
| LAN 側の機器を forward という？ | **違う。** forward は Router VM のカーネルの中にある関所の名前。LAN 側の機器を守るルールを**置く場所**が forward |
| LAN → WAN と WAN → LAN はどう区別する？ | ルーターは LAN・WAN という言葉を知らない。**入ってきた NIC と出ていく NIC** で区別する。ens37 から入って ens33 から出れば「LAN → WAN」 |
| 返事と外からの新しい接続はどう区別する？ | **conntrack の表に当てはまる行があるかどうか**で区別する。行があれば `established`（返事）、無ければ `new`（新しい接続）。NIC で見るとどちらも「ens33 から入って ens37 へ」で、まったく同じ |

## 質問70：forward は関所か

はい、**forward は Router VM のカーネルの中の関所**の1つ。Router VM が受け取ったパケットのうち、**自分宛てではなく、ほかの機器へ転送するもの**だけが通る。

```text
                        Router VM のカーネル
 NIC ──→ ① prerouting ──┬──→ ② input ──→ Router VM 自身が受け取る（SSH など）
                        │
                        └──→ ③ forward ──→ ⑤ postrouting ──→ NIC から出ていく
                              （転送するパケット）
```

| 守る対象 | パケットの行き先 | 止める関所（Router VM の中） |
|---|---|---|
| Router VM 自身 | Router VM 宛て | ② input |
| LAN 側の機器（Client VM など） | Router VM を通り抜けて LAN へ | ③ forward |

LAN 側の機器を守る処理も、**Router VM の中**で行う。WAN から Client VM へ行くパケットは必ず Router VM を通り抜けるので、Router VM の forward で落とせば Client VM には届かない。Client VM 自身は何もしていない。

forward を通るのは「WAN → LAN」だけではない。Client VM から Internet へ出ていく「LAN → WAN」のパケットも forward を通る。

## 質問71：LAN → WAN と WAN → LAN をどう区別するか

ルーター自身は「LAN」「WAN」という言葉を知らない。見ているのは、**どの NIC から入ってきたか**と、**どの NIC から出ていくか**の2つだけ。

| 人間の呼び方 | ルーターが見ているもの |
|---|---|
| LAN → WAN | **ens37 から入って、ens33 から出ていく** |
| WAN → LAN | **ens33 から入って、ens37 から出ていく** |

「ens37 は LAN 側、ens33 は WAN 側」というのは、このラボで人間が決めた意味付け。

### 2つの情報はどこで決まるか

- **入ってきた NIC**：パケットが届いた NIC。届いた時点でわかる
- **出ていく NIC**：ルーティング判断で、宛先 IP とルーティングテーブルを照らし合わせて決まる

Router VM で次のように確かめられる。

```bash
ip route get 8.8.8.8      # → ... dev ens33 ...（default の行に当たる）
ip route get 10.0.0.10    # → ... dev ens37 ...（10.0.0.0/24 の行に当たる）
```

| パケット | 届いた NIC | 宛先 | ルーティングで決まる出口 | 人間の呼び方 |
|---|---|---|---|---|
| Client VM → 8.8.8.8 | ens37 | 8.8.8.8 | ens33 | LAN → WAN |
| 8.8.8.8 → Client VM（返事） | ens33 | 10.0.0.10（prerouting で NAT が書き戻した後） | ens37 | WAN → LAN |

### 関所ごとに、わかる情報が違う

| 関所 | 入ってきた NIC | 出ていく NIC |
|---|---|---|
| ① prerouting | わかる | **まだわからない**（ルーティング判断の前） |
| ② input | わかる | なし（Router VM 自身が受け取るので、出ていかない） |
| ③ forward | わかる | **わかる** |
| ④ output | なし（Router VM 自身が作ったので、入ってきていない） | わかる |
| ⑤ postrouting | わかる（転送の場合） | わかる |

③ forward では両方がそろっているので、向きを区別できる。nftables では次のように書く。

```text
iifname "ens37" oifname "ens33" accept    # LAN → WAN を許可
```

`iifname` は入ってきた NIC、`oifname` は出ていく NIC の名前。labs/04 のマスカレードで使った `oifname "ens33"` も、同じ「出口の NIC」を条件にしている。

### IP アドレスで区別しないのか

`ip saddr 10.0.0.0/24`（送信元が LAN のアドレス）のように IP で書くこともできる。ただ、ファイアウォールでは **NIC で区別するのが基本**。送信元 IP はパケットを送る側が自由に書けるので、WAN 側から `10.0.0.x` になりすましたパケットが届くこともある。どの NIC に届いたかは偽れない。

## 質問72：返事と、外からの新しい接続をどう区別するか

区別しているのは **conntrack**。やっていることは「**記録の表に載っているかどうか**」を見るだけ。

### 仕組み：パケットが届くたびに、表を引く

conntrack は、Router VM を通る通信を1行ずつ表に記録している。パケットが届くと、① prerouting で次の判定をする。

```text
 パケットが届く
     │
     ▼
 conntrack の表に、このパケットに当てはまる行はあるか？
     │
     ├─ ある（その行の「後半＝返事の向き」に一致した）  →  established（返事）
     │
     └─ ない                                           →  new（新しい接続）
                                                         新しい行を作り始める
```

判定の結果はパケットにラベルとして付けられ、そのあとの ③ forward ではラベルを見るだけ。`nft list hooks` で conntrack が prerouting の priority -200 に登録されていたのはこのためで、**forward に着く前に、返事か新しい接続かはもう決まっている**。

### このラボの例で比べる

**Client VM が始めた ping の返事**

Client VM が `ping 8.8.8.8` を打つと、行きのパケットが通ったときに次の1行ができる。

```text
 前半（行きの向き）：src=10.0.0.10  dst=8.8.8.8         type=8 id=1234
 後半（返事の向き）：src=8.8.8.8    dst=192.168.22.129  type=0 id=1234
```

8.8.8.8 から届く返事は `src=8.8.8.8 dst=192.168.22.129 id=1234`。これが**後半とぴったり一致する**ので、`established` と判定される。

**外から始まった新しい接続**

たとえば WAN 側の機器（VMnet8 にいる Windows など）が、Client VM の SSH に直接つなごうとして `10.0.0.10:22` 宛てのパケットを送ったとする。

```text
 表の中： src=?  dst=10.0.0.10  dport=22 … に当てはまる行は無い
```

当てはまる行が無いので、`new` と判定される。

| | 返事 | 外からの新しい接続 |
|---|---|---|
| 届いた NIC | ens33 | ens33 |
| 向き（NIC で見たとき） | WAN → LAN | WAN → LAN |
| 表に当てはまる行 | **ある**（Client VM が行きを送ったときに作られた） | **ない** |
| conntrack の判定 | `established` | `new` |

NIC だけを見ると、2つはまったく同じ。違いは「**先に内側から行きのパケットが出ていたかどうか**」だけで、それを覚えているのが conntrack の表。

### nftables での書き方

```text
chain forward {
	type filter hook forward priority filter; policy drop;

	ct state established,related accept      # ① 返事は通す
	iifname "ens37" oifname "ens33" accept   # ② LAN → WAN の新しい接続は通す
	                                         # どれにも当たらなければ policy drop
}
```

| パケット | 判定 | 当たる rule | 結果 |
|---|---|---|---|
| Client VM → 8.8.8.8（最初の行き） | `new` | ② | 通る（同時に表に行ができる） |
| 8.8.8.8 → Client VM（返事） | `established` | ① | 通る |
| WAN の機器 → 10.0.0.10:22（新しい接続） | `new` | なし（① は established だけ、② は ens37 から来たものだけ） | **捨てる** |

「内側から始めた通信の返事だけが入ってこられる」という家庭用ルーターの基本の動きは、この3行でできている。

### ct state の種類

| 状態 | 意味 | 例 |
|---|---|---|
| `new` | 表に無い。新しい通信の最初のパケット | Client VM が初めて送る ping |
| `established` | 表にあり、その通信の続きか返事 | ping の返事、SSH のやりとり |
| `related` | 既存の通信に**関係する**、別のパケット | 「宛先に届かない」と知らせる ICMP のエラー |
| `invalid` | どれにも当てはまらない、おかしなパケット | 表に無い TCP 接続の途中のパケット |

### 補足

**なりすましは難しい**

外から「返事のふり」をするには、表の後半と同じ IP・ポート（ping なら id）を当てる必要がある。ポートは毎回ランダムに選ばれるので、当てるのは難しい。TCP ではさらに、シーケンス番号が正しい範囲にあるかも確かめられる。

**記録には期限がある**

しばらく通信が無いと、行は消える（おおよそ ICMP は 30 秒、TCP の確立済み接続は 5 日）。行が消えたあとに届いた「返事」は `new` 扱いになり、上の rule では捨てられる。

**NAT だけでは守れない理由**

上の「WAN の機器 → 10.0.0.10:22」は、WAN 側の機器に「10.0.0.0/24 へは 192.168.22.129 経由」というルートを足せば送れる。NAT（masquerade）は**出ていくパケット**の送信元を書き換えるだけで、入ってくる `new` のパケットは止めない。今の Router VM は forward に rule を何も置いていないので、このパケットは Client VM まで届くはず。だからファイアウォールが必要になる（labs/05 で確かめる）。

### labs/05 で確かめられること

- `sudo conntrack -E` を動かしたまま Client VM から ping を打ち、行ができる瞬間（`[NEW]`）を見る
- rule に `counter` を付け、`sudo nft list ruleset` でどの rule に何個のパケットが当たったかを数える
- WAN 側からの新しい接続が、rule を入れる前は届き、入れた後は止まることを確かめる
