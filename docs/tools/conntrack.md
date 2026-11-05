# conntrack コマンド

カーネルの conntrack の表（通信の記録）を、見たり操作したりするコマンドです。conntrack の仕組みは [docs/concepts/conntrack.md](../concepts/conntrack.md) を参照してください。

## インストール

Debian には最初から入っていません。`conntrack` パッケージに入っています。

```bash
sudo apt update
sudo apt install conntrack
```

- `nft` と同じく `/usr/sbin` に入るので、**`sudo` を付けて実行する**
- コマンドを入れなくても、カーネルの conntrack 自体は動いている。コマンドは記録を**見る**ためのもの

---

## よく使う使い方

| コマンド | 意味 |
|---|---|
| `sudo conntrack -L` | 今ある記録を一覧で表示する（**L**ist） |
| `sudo conntrack -E` | 記録が作られたり、消えたりするたびに表示し続ける（**E**vent）。`Ctrl + C` で止める |
| `sudo conntrack -C` | 記録の数だけを表示する（**C**ount） |
| `sudo conntrack -D …` | 条件に合う記録を消す（**D**elete） |
| `sudo conntrack -F` | 記録を全部消す（**F**lush） |

### 条件で絞る

`-L` や `-D` には、条件を付けられます。

| 条件 | 意味 |
|---|---|
| `-p icmp` | プロトコルが ICMP の記録 |
| `-s 10.0.0.10` | 前半（行きの向き）の送信元が `10.0.0.10` |
| `-d 8.8.8.8` | 前半（行きの向き）の宛先が `8.8.8.8` |

例：

```bash
sudo conntrack -L -p icmp
sudo conntrack -L -s 10.0.0.10
```

---

## 記録を見るときの注意

ICMP の記録は、通信が終わってから**約30秒で消えます**。ping を止めてから `-L` を実行すると、もう消えていることがあります。

Client VM で回数を指定せずに ping を送り続け、その間に Router VM で見ます。

```bash
ping 8.8.8.8            # Client VM。終わったら Ctrl + C
```

```bash
sudo conntrack -L       # Router VM
```

`-E` を動かしたまま Client VM から ping を始めると、記録が**作られる瞬間**と、ping を止めて約30秒後に**消える瞬間**を見られます。

---

## 表示の読み方

Client VM から ping を送り続けている間に、Router VM で `sudo conntrack -L` を実行した結果の1行（2026-11-05、labs/04）：

```text
icmp     1 29 src=10.0.0.10 dst=8.8.8.8 type=8 code=0 id=14 src=8.8.8.8 dst=192.168.22.129 type=0 code=0 id=14 mark=0 use=1
              └──────────── 前半（行きの向き）─────────────┘ └──────────── 後半（帰りの向き）──────────────┘
```

| 部分 | 意味 |
|---|---|
| `icmp` `1` | プロトコルの名前と番号（TCP は `6`、UDP は `17`） |
| `29` | 記録が消えるまでの残り秒数 |
| 前半の `src=10.0.0.10 dst=8.8.8.8` | 行きのパケットの、書き換える前の送信元と宛先 |
| 後半の `src=8.8.8.8 dst=192.168.22.129` | 帰りに届くはずの返事。宛先が、書き換えた後の `192.168.22.129` |
| `type=8` / `type=0` | ICMP の種類。8 は echo request、0 は echo reply |
| `id=14` | ICMP の id。TCP/UDP では、代わりに `sport=`（送信元ポート）と `dport=`（宛先ポート）が出る |

後半が、前半をそのまま逆にした形（`src=8.8.8.8 dst=10.0.0.10`）と違っていれば、その通信は NAT されています。

TCP の行には、ほかに次のものが出ます。

```text
tcp      6 431999 ESTABLISHED src=192.168.22.1 dst=192.168.22.129 sport=53618 dport=22 src=192.168.22.129 dst=192.168.22.1 sport=22 dport=53618 [ASSURED] mark=0 use=1
```

| 部分 | 意味 |
|---|---|
| `ESTABLISHED` | TCP の接続の状態。やりとりが続いている |
| `431999` | 残り秒数。`ESTABLISHED` の TCP は、5日間（432,000 秒）通信が無いと消える |
| `[ASSURED]` | 両方向にやりとりがあった、確かな通信という印 |

conntrack は、転送する通信だけでなく、Router VM 自身の通信（DHCP や、Router VM から Client VM への SSH）や、Router VM 宛ての通信（Windows からの SSH）も記録します。
