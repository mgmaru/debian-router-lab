# NIC 名（`ens33`）と、1 つの NIC に付けられる IP アドレスの数

調査日：2026-10-11

関連：

- [`ip route` とルーティングテーブルとは何か](2026-10-11-what-is-ip-route-and-routing-table.md)（質問1〜3）
- [docs/concepts/interface.md（`ip a` の読み方）](../concepts/interface.md#ip-a-の読み方)
- [labs/01-network-interface](../../labs/01-network-interface/README.md)

## 疑問

> **質問4（追加質問）**：`ip link` は NIC を表しているとのことだが、NIC 名とは `ens33` などのことか？

> **質問5（追加質問）**：NIC には IP アドレスをいくつまで付けられる？追加方法は？

## 結論

| 質問 | 答え |
|---|---|
| NIC 名は `ens33` など？ | **その通り。** `ens33` は、Linux が NIC に付けた**インターフェース名**である。Router VM なら `ens33`（WAN）と `ens37`（LAN）。ただし `ip link` には、`lo` のような **NIC ではないインターフェース**も出る |
| いくつまで付けられる？ | **決まった上限は、実質ない。** 1 つの NIC に複数の IP を付けるのは普通のことで、このラボの Client VM にも一時期 2 つ付いていた |
| 追加方法は？ | 一時的なら `sudo ip addr add <IP>/<長さ> dev <NIC>`。永続化するなら、`/etc/network/interfaces` に同じ NIC の `iface` の段落をもう 1 つ書く |

---

## 質問4：NIC 名とは `ens33` などのことか

### NIC とインターフェース

| 言葉 | 意味 | このラボでは |
|---|---|---|
| NIC | ネットワークにつなぐための部品（ハードウェア） | VMware が VM に差した仮想 NIC |
| インターフェース | Linux が NIC を扱うための窓口。名前が付いている | `ens33`、`ens37` |

`ip link` が一覧表示しているのは**インターフェース**である。NIC 1 枚につきインターフェースが 1 つできるので、普段は「NIC 名 ≒ インターフェース名」と考えてよい。

### `ip link` の表示（Router VM の例）

```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ... state UNKNOWN ...
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ... state UP ...
    link/ether 00:0c:29:73:2d:61 brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    altname enx000c29732d61
3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ... state UP ...
    link/ether 00:0c:29:73:2d:6b brd ff:ff:ff:ff:ff:ff
```

（labs/01 で `ip a` を実行したときの内容をもとにした例。`ip link` そのものはまだ実行していない）

| 表示 | 意味 |
|---|---|
| 先頭の番号（`2:`） | インターフェース番号。Router VM の `ip route` に出た `metric 1002` の `2` はこれ |
| `ens33` | インターフェース名。設定ファイルにはこの名前を書く |
| `UP` / `state UP` | インターフェースが動いている |
| `mtu 1500` | 一度に送れるパケットの最大サイズ（バイト） |
| `link/ether ...` | MAC アドレス |
| `altname` | 同じインターフェースの別名 |

**`ip link` には IP アドレスが出ない。** `ip addr` は「`ip link` の内容 ＋ IP アドレス」を表示する、と考えるとよい。

### `lo` は NIC ではない

`lo`（ループバック）は、**自分自身と通信するための仮想インターフェース**である。`127.0.0.1` が付いていて、ケーブルの先はどこにもつながっていない。

このように、`ip link` には NIC 以外のインターフェースも並ぶ。今後 VLAN や VPN を作ると、それも仮想インターフェースとして増える（labs/09、labs/10）。

### `ens33` という名前の意味

| 部分 | 意味 |
|---|---|
| `en` | Ethernet（有線 LAN） |
| `s33` | スロット（slot）番号 33。VMware がこの仮想 NIC を差したスロットの番号 |

`ens37` は「Ethernet、スロット 37」である。

`altname` に出ていた別名も、同じ NIC に別のルールで付けた名前である。

| 名前 | 何をもとにした名前か |
|---|---|
| `ens33` | スロット番号（33） |
| `enp2s1` | PCI 上の場所（バス 2、スロット 1） |
| `enx000c29732d61` | MAC アドレス（`00:0c:29:73:2d:61`） |

> 💡 昔の Linux は、見つけた順に `eth0`、`eth1` と名前を付けていた。この順番が起動のたびに入れ替わることがあり、`eth0` が WAN になったり LAN になったりして困った。そこで今は、ハードウェアの場所などから**毎回同じ名前になる**付け方（Predictable Network Interface Names）が使われている。

---

## 質問5：NIC に IP アドレスはいくつ付けられるか

### いくつまで付けられるか

**決まった上限は、実質ない。** 1 つの NIC に複数の IP アドレスを付けるのは普通のことである。

このラボにも実例がある。labs/01 で、Client VM の ens33 には 2 つのアドレスが同時に付いていた。

```text
inet 169.254.165.204/16 brd 169.254.255.255 scope global noprefixroute ens33
inet 10.0.0.10/24 scope global ens33
```

DHCP クライアントが付けたリンクローカルアドレスと、手で付けたアドレスである（[interface.md](../concepts/interface.md#リンクローカルアドレス169254xx)）。

使いどころの例：

- 同じケーブルの先に、別のネットワーク（例：`10.0.1.0/24`）も置きたいとき
- 1 台のサーバーで、サービスごとに別の IP を使いたいとき
- IP を変えるときに、古い IP と新しい IP をしばらく両方付けておきたいとき

> 💡 IPv6 では、1 つの NIC に複数のアドレス（`fe80::` で始まるリンクローカルアドレスなど）が付くのが普通である（labs/08）。

### 追加方法（一時的）

```bash
sudo ip addr add 10.0.0.20/24 dev ens33   # 追加（今ある 10.0.0.10 は消えない）
ip addr show ens33                         # 確認
sudo ip addr del 10.0.0.20/24 dev ens33   # 削除
```

追加すると、`ip addr show ens33` に次のような行が増える（予想）。

```text
inet 10.0.0.10/24 ... scope global ens33
inet 10.0.0.20/24 scope global secondary ens33
```

| 表示 | 意味 |
|---|---|
| `secondary` | 同じネットワーク（`10.0.0.0/24`）の 2 つ目以降のアドレスに付く印 |

**ルーティングテーブルとの関係：**

| 追加したアドレス | ルーティングテーブル |
|---|---|
| `10.0.0.20/24`（今と同じネットワーク） | 行は増えない。`10.0.0.0/24` の行がすでにあるから |
| `10.0.1.1/24`（別のネットワーク） | `10.0.1.0/24 dev ens33 proto kernel ...` の行が増える |

アドレスが付くとカーネルが自動で経路を作る、という[質問2](2026-10-11-what-is-ip-route-and-routing-table.md#各行は誰が作っているか)の話と同じである。

> ⚠️ 注意
>
> - `/24` を付け忘れると `/32`（自分 1 台だけのネットワーク）として付き、意図しない設定になる
> - SSH で使っている IP を `del` すると、接続が切れる
> - `ip addr add` は一時的な設定なので、再起動すると消える

### 追加方法（永続化）

Debian の `/etc/network/interfaces` では、**同じ NIC の `iface` の段落をもう 1 つ書く**。設定ファイルの説明（`man 5 interfaces`）にも「同じインターフェースに `iface` の段落を複数書くと、すべてのアドレスが適用される」とある。

```text
auto ens33
iface ens33 inet static
    address 10.0.0.10/24
    gateway 10.0.0.1

iface ens33 inet static
    address 10.0.0.20/24
```

`gateway` は 1 つ目の段落にだけ書く（デフォルトゲートウェイは基本的に 1 つ）。

`up` の行で追加する書き方もある（Debian Wiki の例）。

```text
iface ens33 inet static
    address 10.0.0.10/24
    gateway 10.0.0.1
    up   ip addr add 10.0.0.20/24 dev $IFACE
    down ip addr del 10.0.0.20/24 dev $IFACE
```

`$IFACE` は、ifupdown が NIC 名（ここでは `ens33`）に置き換えてくれる変数である。

※ どちらも書き方の例である。このラボではまだ必要ない。

> 💡 ネットでは `ens33:0` のような書き方（エイリアス）を見かけることがある。`ifconfig` の時代の古い方法で、Debian Wiki でも非推奨とされている。今は上の方法で、同じ NIC に直接追加すればよい。

---

## まだ試していないこと

| コマンド | 予想される結果 | 確かめられること |
|---|---|---|
| （Router VM）`ip link` | `lo`、`ens33`、`ens37` の 3 つが出る | NIC と、NIC ではない `lo` が並ぶ |
| （Router VM）`ip -br link` | 1 行に 1 インターフェースの短い表示が出る | 一覧を素早く見る方法（`-br` は brief の略） |
| （Client VM）`sudo ip addr add 10.0.0.20/24 dev ens33` → `ip addr show ens33` | `inet 10.0.0.20/24 scope global secondary ens33` が増える | 1 つの NIC に複数の IP を付けられる |
| （Router VM）`ping -c 2 10.0.0.20` | 応答が返る | 追加した IP でも通信できる |
| （Client VM）`ip route` | 行は増えていない | 同じネットワークのアドレスなら、経路は増えない |
| （Client VM）`sudo ip addr del 10.0.0.20/24 dev ens33` | `10.0.0.20` の行が消える | 一時的に追加した IP の消し方 |

---

## まとめ

- `ens33` は、Linux が NIC に付けた**インターフェース名**。`ip link` はインターフェースの一覧で、`lo` のような NIC ではないものも並ぶ。
- `ens33` は「**Ethernet、スロット 33**」という意味。毎回同じ名前になるように、ハードウェアの場所から付けられている。
- 1 つの NIC に付けられる IP アドレスに、**決まった上限は実質ない**。複数付けるのは普通のこと。
- 一時的に追加するなら `sudo ip addr add <IP>/<長さ> dev <NIC>`。永続化するなら、`/etc/network/interfaces` に**同じ NIC の `iface` の段落をもう 1 つ**書く。
