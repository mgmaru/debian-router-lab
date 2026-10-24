# `ip link` の `BROADCAST,MULTICAST` は、ブロードキャストやマルチキャストを受け付けるという意味か

調査日：2026-10-24

まとまった説明：[docs/concepts/interface.md](../concepts/interface.md)

関連：

- [NIC 名（`ens33`）と、1 つの NIC に付ける IP アドレス（数・ネットワーク）](2026-10-11-nic-name-and-multiple-ip-addresses.md)（質問4：`ip link` の表示）
- [ネットワークアドレスとブロードキャストアドレス：`.0` と `.255` の意味と使い道](2026-10-13-network-address-in-routing-table.md)（IP と MAC のブロードキャストアドレス）
- [DHCP サーバーのアドレスはどこで分かるのか](2026-10-11-where-to-find-dhcp-server-address.md)（`ifflags=69699`）
- [labs/01-network-interface（Experiment 7：`ip link` の観察）](../../labs/01-network-interface/README.md#7-インターフェースの観察router-vm--client-vm)

## 疑問

> **質問33**：`ip link` で、インターフェースごとに `<BROADCAST,MULTICAST,UP,LOWER_UP>` のような設定が表示される。これは、マルチキャストやブロードキャストを受け付けるという意味か？

## 結論

| 質問 | 答え |
|---|---|
| `BROADCAST` は「ブロードキャストを受け付ける」という意味？ | **違う。「このインターフェースは、ブロードキャストという仕組みに対応している」という印**である。Ethernet には全員宛ての MAC アドレス（`ff:ff:ff:ff:ff:ff`）があるので、必ず付く。受け付けるかどうかを切り替えるスイッチではない |
| `MULTICAST` は「マルチキャストを受け付ける」という意味？ | **「マルチキャストを使える」という意味**である。どのグループ宛てのパケットを実際に受け取るかは、別の一覧（`ip maddr`）で管理されている |
| `< >` の中は何か | **インターフェースの状態を表すフラグ**（印）の一覧。種類で決まるもの（`BROADCAST`、`LOOPBACK`）と、状態で変わるもの（`UP`、`LOWER_UP`）が混ざっている |

---

## `< >` の中のフラグ

Router VM の `ip link`（2026-10-24 に実行）：

```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:73:2d:61 brd ff:ff:ff:ff:ff:ff
```

| フラグ | 意味 | 何で決まるか |
|---|---|---|
| `BROADCAST` | ブロードキャストに対応している（ブロードキャストアドレスを持てる） | インターフェースの種類。Ethernet なら付く |
| `MULTICAST` | マルチキャストを使える | 通常はオン。`ip link set <NIC> multicast off` で切れる |
| `LOOPBACK` | 自分自身に戻すだけのインターフェース | `lo` だけに付く |
| `UP` | 管理者が「使う」と設定している | `ip link set <NIC> up` / `down` |
| `LOWER_UP` | ケーブル（仮想ケーブル）がつながっていて、物理的に通信できる | 接続状態。ケーブルが抜けると消え、`NO-CARRIER` が出る |

`BROADCAST` と `MULTICAST` は「何ができるか」、`UP` と `LOWER_UP` は「今どうなっているか」を表している、と分けて考えるとよい。

---

## `BROADCAST`：仕組みがあるかどうか

Ethernet には、**全員宛ての MAC アドレス `ff:ff:ff:ff:ff:ff`** がある。`ip link` の 2 行目に出ている `brd ff:ff:ff:ff:ff:ff` がそれである。

```text
2: ens33: <BROADCAST,...>
    link/ether 00:0c:29:73:2d:61 brd ff:ff:ff:ff:ff:ff
                                 ↑ ブロードキャストアドレス
```

`BROADCAST` は、「このインターフェースには、こういう全員宛てのアドレスがある」という印である。Linux の説明（`man 7 netdevice`）でも、`IFF_BROADCAST` は「ブロードキャストアドレスが有効」と書かれている。

Ethernet の NIC は、`ff:ff:ff:ff:ff:ff` 宛てのフレームを普通に受け取る。ARP の問い合わせや DHCP の Discover が届くのは、このためである（[質問23](2026-10-13-network-address-in-routing-table.md#質問231000255-はネットワーク内の全員に対する宛先か)）。**「ブロードキャストを受け取らない」ように切り替えるフラグではない**。

### `lo` に付いていない理由

```text
1: lo: <LOOPBACK,UP,LOWER_UP>
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP>
```

`lo` は自分自身にしかつながっていない。「全員に送る」相手がいないので、ブロードキャストという仕組み自体がない。`lo` の `brd 00:00:00:00:00:00` も、意味のある全員宛てのアドレスではない。

同じ `ip link` の中で、Ethernet（ens33）とループバック（lo）のフラグが違うことから、**フラグはインターフェースの種類による性質**を表していると分かる。

---

## `MULTICAST`：使えるかどうか

マルチキャストは、**参加しているグループのメンバーにだけ届ける**送り方である。

| 送り方 | 宛先 |
|---|---|
| ユニキャスト | 1 台だけ |
| ブロードキャスト | 同じネットワークの全員 |
| マルチキャスト | グループに参加している機器だけ |

`MULTICAST` フラグは、「このインターフェースはマルチキャストを使える」という印である。実際にどのグループ宛てのパケットを受け取るかは、フラグとは別に、**参加しているグループの一覧**で管理されている。

```bash
ip maddr     # 参加しているマルチキャストグループの一覧
```

IPv6 では、ARP の代わりの仕組み（近隣探索）にマルチキャストを使う。そのため、普段は `MULTICAST` を切らない。

---

## フラグは数字でも表せる

[DHCP サーバーのアドレスはどこで分かるのか](2026-10-11-where-to-find-dhcp-server-address.md) で見た `sudo dhcpcd -U ens33` の出力に、`ifflags=69699` という行があった。これは、**同じフラグを 1 つの数字にまとめたもの**である。

カーネルの中では、フラグはそれぞれ 1 ビットで表されている。

| フラグ | 値 |
|---|---|
| `UP` | 1 |
| `BROADCAST` | 2 |
| `RUNNING`（`ip link` には出ない。`LOWER_UP` に近い意味） | 64 |
| `MULTICAST` | 4,096 |
| `LOWER_UP` | 65,536 |
| **合計** | **69,699** |

つまり `ifflags=69699` は、`<BROADCAST,MULTICAST,UP,LOWER_UP>` を数字で書いたものである。`ip link` は、この数字を読みやすい名前に直して表示している。

---

## まだ試していないこと

| コマンド | 予想される結果 | 確かめられること |
|---|---|---|
| （Router VM）`ip maddr show ens37` | `01:00:5e:00:00:01` や `33:33:00:00:00:01` などのマルチキャストアドレスが並ぶ | `MULTICAST` フラグとは別に、参加しているグループの一覧がある |
| （Router VM）`ip -br link` | `ens37  UP  00:0c:29:73:2d:6b <BROADCAST,MULTICAST,UP,LOWER_UP>` のように 1 行で出る | フラグは簡潔な表示でも見られる |
| （VMware）Client VM の NIC の「接続済み」を外す → （Client VM）`ip link show ens33` | `<NO-CARRIER,BROADCAST,MULTICAST,UP>` になり、`LOWER_UP` が消えて `state DOWN` になる | `UP`（設定）と `LOWER_UP`（接続状態）は別物。`BROADCAST` と `MULTICAST` は接続状態では変わらない |

---

## まとめ

- `< >` の中は、インターフェースの状態を表す**フラグ**の一覧である。
- `BROADCAST` は「ブロードキャストを受け付ける」ではなく、**「ブロードキャストという仕組みに対応している」**という印。Ethernet には `ff:ff:ff:ff:ff:ff` があるので付き、`lo` には付かない。
- `MULTICAST` は**「マルチキャストを使える」**という印。実際にどのグループ宛てを受け取るかは `ip maddr` で管理されている。
- `BROADCAST` / `MULTICAST` は「何ができるか」、`UP` / `LOWER_UP` は「今どうなっているか」を表す。
- `dhcpcd -U` の `ifflags=69699` は、同じフラグを数字で表したものである。

---

## 参考

- [netdevice(7)：Linux のインターフェースのフラグ（`IFF_BROADCAST` など）](https://man7.org/linux/man-pages/man7/netdevice.7.html)
- [ip-link(8)](https://man7.org/linux/man-pages/man8/ip-link.8.html)
- [ip-maddress(8)](https://man7.org/linux/man-pages/man8/ip-maddress.8.html)
