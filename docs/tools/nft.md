# nft コマンド

nftables を操作するコマンドです。table・chain・rule や関所（フック）の考え方は [docs/concepts/kernel/nftables.md](../concepts/kernel/nftables.md) を参照してください。

## 実行するときの注意

- **いつも `sudo` を付ける**。`nft` は `/usr/sbin/nft` にあり、一般ユーザーの PATH に無いので、`sudo` を付けないと `command not found` になる
- コマンドで入れた設定は、**再起動すると消える**。永続化は `/etc/nftables.conf` に書く（下の「ファイルから読み込む」）

---

## 基本の形

```text
sudo nft <操作> <対象> <ファミリー> <table 名> [<chain 名>] [<中身>]
```

| 部分 | 書くもの | 例 |
|---|---|---|
| 操作 | 何をするか | `add`（作る・足す）、`list`（見る）、`delete`（消す）、`flush`（中身を空にする） |
| 対象 | 何に対してか | `table`、`chain`、`rule`、`ruleset`（全部） |
| ファミリー | どの種類のパケットの table か | `ip`（IPv4）、`ip6`（IPv6）、`inet`（両方） |
| table 名 | どの table か | `nat` |
| chain 名 | どの chain か（chain・rule のとき） | `postrouting` |
| 中身 | chain の設定や、rule の条件と動作 | `oifname "ens33" masquerade` |

外側（table）から順に名前を指定していく、と覚えると迷いません。

```text
table を作る：  sudo nft add table  ip nat
chain を作る：  sudo nft add chain  ip nat postrouting '{ … }'
rule を足す：   sudo nft add rule   ip nat postrouting oifname "ens33" masquerade
                                    └┬┘└┬┘└────┬────┘
                              ファミリー table   chain
```

---

## 作る・足す（add）

### table を作る

```text
sudo nft add table <ファミリー> <table 名>
```

```bash
sudo nft add table ip nat
```

- 同じファミリーに同じ名前の table がすでにあると、`add` はエラーにならず何もしない（すでにある table をそのまま使う）
- すでにあるときにエラーにしたい場合は、`add` の代わりに `create` を使う（`File exists` になる）

### chain を作る

```text
sudo nft add chain <ファミリー> <table 名> <chain 名> '{ type <種類> hook <関所> priority <順番>; policy <動作>; }'
```

```bash
sudo nft add chain ip nat postrouting '{ type nat hook postrouting priority srcnat; policy accept; }'
```

| 部分 | 今回の値 | 意味 |
|---|---|---|
| `<chain 名>` | `postrouting` | chain の**名前**。自由に付けられる。関所と同じ名前にするのが習慣 |
| `type <種類>` | `nat` | 何用の chain か。`filter`（通す・捨てる）、`nat`（書き換え）、`route` |
| `hook <関所>` | `postrouting` | どの関所に置くか。**ここが実際の関所の指定** |
| `priority <順番>` | `srcnat` | 同じ関所での順番。小さいほど先 |
| `policy <動作>` | `accept` | どの rule にも当てはまらないときの動作。`accept` か `drop` |

- `<chain 名>` と `hook <関所>` は、どちらも `postrouting` なので紛らわしい。前者はただの名前、後者が実際に置く関所
- `{ }` の中の区切りは `;`

priority は、数字の代わりに名前でも書けます。

| 名前 | 数字 | 主な使い道 |
|---|---|---|
| `dstnat` | -100 | 宛先の書き換え（prerouting に置く） |
| `filter` | 0 | 通す・捨てる |
| `srcnat` | 100 | 送信元の書き換え（postrouting に置く） |

### rule を足す

```text
sudo nft add rule <ファミリー> <table 名> <chain 名> <条件> <動作>
```

```bash
sudo nft add rule ip nat postrouting oifname "ens33" masquerade
```

- `add rule` は、chain の**いちばん下**に足す
- `insert rule` は、chain の**いちばん上**に足す。rule は上から順に見られるので、先に見てほしい rule は `insert` で入れる

`oifname "ens33" masquerade` の詳しい意味（条件が必要な理由など）は、[docs/concepts/kernel/nftables.md](../concepts/kernel/nftables.md) の「`oifname "ens33" masquerade` の意味」を参照してください。

よく使う条件と動作：

| 条件 | 意味 |
|---|---|
| `oifname "ens33"` | 出ていく NIC が ens33（**o**utput **i**nter**f**ace **name**） |
| `iifname "ens37"` | 入ってきた NIC が ens37（**i**nput **i**nter**f**ace **name**） |
| `ip saddr 10.0.0.0/24` | 送信元 IP が `10.0.0.0/24` の中 |
| `ip daddr 8.8.8.8` | 宛先 IP が `8.8.8.8` |
| `tcp dport 22` | 宛先ポートが 22（SSH） |
| `ct state established,related` | すでに始まっている通信（conntrack の記録がある）のパケット |

| 動作 | 意味 |
|---|---|
| `masquerade` | 送信元を、出ていく NIC の IP に書き換える |
| `snat to <IP>` | 送信元を、指定した IP に書き換える（WAN 側の IP が固定のとき）。`masquerade` は IP をその都度自動で調べるので、DHCP で IP が変わる WAN 側に向いている |
| `accept` | 通す |
| `drop` | 黙って捨てる |
| `counter` | 当てはまったパケットの数を数える（ほかの動作と一緒に書ける。例：`counter masquerade`） |

---

## 見る（list）

| コマンド | 表示するもの |
|---|---|
| `sudo nft list ruleset` | すべての table・chain・rule（入れ子の形） |
| `sudo nft list tables` | table の一覧だけ |
| `sudo nft list table ip nat` | 1つの table の中身 |
| `sudo nft list chain ip nat postrouting` | 1つの chain の中身 |
| `sudo nft -a list ruleset` | rule ごとに番号（`# handle 3` など）も表示する。rule を1つだけ消すときに使う |
| `sudo nft list hooks` | 関所ごとに、何が登録されているか（カーネル自身が登録したものも出る） |

`list ruleset` と `list hooks` の違いは、[docs/concepts/kernel/nftables.md](../concepts/kernel/nftables.md) の「`list ruleset` と `list hooks` の違い」を参照してください。

---

## 消す（delete・flush）

| コマンド | 消すもの |
|---|---|
| `sudo nft delete rule ip nat postrouting handle 3` | rule を1つ（番号は `nft -a list ruleset` で調べる） |
| `sudo nft flush chain ip nat postrouting` | chain の中の rule を全部（chain は残る） |
| `sudo nft delete chain ip nat postrouting` | chain（中に rule があると消せない。先に `flush chain` する） |
| `sudo nft delete table ip nat` | table を、中の chain・rule ごと全部 |
| `sudo nft flush ruleset` | **すべての table**。ほかのソフトが入れた table も消える |

rule は、名前ではなく **handle（番号）** で指定して消します。

```text
kajih@debian-router:~$ sudo nft -a list ruleset
table ip nat { # handle 1
	chain postrouting { # handle 1
		type nat hook postrouting priority srcnat; policy accept;
		oifname "ens33" masquerade # handle 2
	}
}
```

上の例なら、`sudo nft delete rule ip nat postrouting handle 2` で消せます（番号は環境によって変わります）。

---

## ファイルから読み込む（永続化）

`/etc/nftables.conf` などのファイルに、`list ruleset` と同じ入れ子の形で書いて読み込みます。

| コマンド | 意味 |
|---|---|
| `sudo nft -c -f /etc/nftables.conf` | 書き方が正しいかだけを確かめる（読み込まない） |
| `sudo nft -f /etc/nftables.conf` | ファイルを読み込む |
| `sudo systemctl enable --now nftables` | 起動時に `/etc/nftables.conf` を読み込むようにし、今すぐ読み込む |

`sudo nft list ruleset` の出力は、そのままファイルの書き方として使えます。コマンドで入れて確かめた後に、出力をファイルに写すと間違いにくくなります。

---

## よくある間違い

| 症状 | 原因 | 直し方 |
|---|---|---|
| `nft: command not found` | `sudo` を付けていない | `sudo nft …` で実行する |
| `>` が出て、コマンドが終わらない | `'...'` の閉じの `'` を付け忘れた。シェルが続きの入力を待っている | `Ctrl + C` で取り消して、`'` を付けて打ち直す |
| chain の設定がうまく渡らない | `'{ … }'` を `'` で囲んでいない。`;` がシェルのコマンドの区切りとして解釈される | `{ … }` 全体を `'` で囲む |
| `No such file or directory` | 指定した table や chain が無い。名前やファミリー（`ip` か `inet` か）の間違い | `sudo nft list ruleset` で名前を確かめる |
| 同じ rule が2つある | `add rule` を2回実行した。`add rule` は、同じ rule でも重ねて足す | `nft -a list ruleset` で番号を調べて、片方を消す |
