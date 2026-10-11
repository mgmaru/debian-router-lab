# `ip addr` にデフォルトゲートウェイは表示されないのか

調査日：2026-10-11

関連：

- [docs/concepts/interface.md（`ip a` の読み方）](../concepts/interface.md#ip-a-の読み方)
- [docs/concepts/routing.md（デフォルトゲートウェイ）](../concepts/routing.md#デフォルトゲートウェイ)
- [デフォルトゲートウェイとは「同じネットワークで転送先が見つからない場合の送り先」か](2026-10-08-default-gateway-meaning.md)

## 疑問

> `ip addr` にデフォルトゲートウェイは表示されないのか？

## 結論

**表示されない。** `ip addr` と `ip route` は見ているものが違う。

| コマンド | 見ているもの | Client VM での例 |
|---|---|---|
| `ip addr` | 各インターフェースに付いているアドレス（MAC、IP/プレフィックス、状態） | `inet 10.0.0.10/24 brd 10.0.0.255 scope global ens33` |
| `ip route` | ルーティングテーブル（宛先ごとの送り方） | `default via 10.0.0.1 dev ens33` |

デフォルトゲートウェイはインターフェースの属性ではなく、**ルーティングテーブルの 1 行（経路）**である。そのため `ip route` で確認する。デフォルトの行だけ見たいときは `ip route show default` を使う。

---

## 紛らわしいところ

### 1. 設定ファイルではインターフェースの段落に書く

Client VM の `/etc/network/interfaces` では、`gateway` を `iface ens33` の段落の中に書いている（[configs/network/debian-client.interfaces](../../configs/network/debian-client.interfaces)）。

```text
iface ens33 inet static
    address 10.0.0.10/24
    gateway 10.0.0.1
```

見た目はインターフェースの設定だが、起動時に `ifup` がこれを `ip route add default via 10.0.0.1 dev ens33` に相当する処理に変えて、ルーティングテーブルに入れる。

**書く場所はインターフェースの段落でも、実際に入るのはルーティングテーブルである。**

### 2. `brd 10.0.0.255` はゲートウェイではない

`ip addr` に出る `brd` はブロードキャストアドレス（同じネットワークの全員宛て）である。ゲートウェイとは関係ない。

### 3. Windows の `ipconfig` とは違う

Windows の `ipconfig` は、アダプタごとに IP アドレスとデフォルトゲートウェイを並べて表示する。それに慣れていると「アドレスと一緒に出るはず」と思いやすい。

---

## `ip addr` と `ip route` のつながり

`ip addr` で `10.0.0.10/24` を付けると、カーネルが次の経路を自動で `ip route` に追加する。

```text
10.0.0.0/24 dev ens33 proto kernel scope link src 10.0.0.10
```

`proto kernel` は「カーネルが自動で作った経路」という印である。

一方、**デフォルトゲートウェイはアドレスからは自動で作られない。** `gateway` 行か DHCP で、別に指定する必要がある。

| VM / NIC | ゲートウェイの入り方 |
|---|---|
| Client VM の ens33（static） | `gateway 10.0.0.1` 行から `default via 10.0.0.1 dev ens33` が入る |
| Router VM の ens33（DHCP） | DHCP が配った `default via 192.168.22.2 dev ens33` が自動で入る |
| Router VM の ens37（LAN 側） | `gateway` を書いていない。Router VM 自身が LAN のゲートウェイだから |

---

## まだ試していないこと

| コマンド（Client VM） | 予想される結果 | 確かめられること |
|---|---|---|
| `ip addr show ens33` | `10.0.0.1` はどこにも出てこない | `ip addr` にはゲートウェイが出ない |
| `ip route show default` | `default via 10.0.0.1 dev ens33` | ゲートウェイはルーティングテーブルに入っている |
| `ip route` | `10.0.0.0/24 dev ens33 proto kernel scope link src 10.0.0.10` の行がある | アドレスを付けると、自分のネットワークへの経路が自動で作られる |

---

## まとめ

- `ip addr` は**インターフェースに付いたアドレス**を見るコマンド。デフォルトゲートウェイは出ない。
- デフォルトゲートウェイは**ルーティングテーブルの 1 行**。`ip route`（または `ip route show default`）で見る。
- `/etc/network/interfaces` の `gateway` はインターフェースの段落に書くが、**入る先はルーティングテーブル**。
- アドレスからは自分のネットワークへの経路（`proto kernel`）だけが自動で作られる。デフォルトゲートウェイは**別に指定が必要**。
