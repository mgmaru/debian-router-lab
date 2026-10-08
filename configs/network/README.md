# Network

Debian（ifupdown）の `/etc/network/interfaces` に配置する設定ファイル。

| ファイル | 配置先 | 内容 |
|---|---|---|
| [debian-router.interfaces](debian-router.interfaces) | debian-router の `/etc/network/interfaces` | ens33（WAN）は DHCP、ens37（LAN）は `10.0.0.1/24` |
| [debian-client.interfaces](debian-client.interfaces) | debian-client の `/etc/network/interfaces` | ens33（LAN）は `10.0.0.10/24`、ゲートウェイは `10.0.0.1` |

反映の手順は [environment/debian/network.md](../../environment/debian/network.md)、ifupdown を使う理由は [006-use-ifupdown.md](../../docs/decisions/006-use-ifupdown.md) を参照。

---

## `/etc/network/interfaces` の書き方

### 各行の意味

```
auto ens37
iface ens37 inet static
    address 10.0.0.1/24
```

| 行 | 意味 |
|---|---|
| `auto ens37` | **起動時に ens37 を自動で立ち上げる**。起動時に `networking` サービスが `ifup -a`（`auto` が付いたものを全部上げる）を実行する。この行がないと、設定が書いてあっても手で `ifup` するまで使われない |
| `iface ens37 inet static` | **ens37 の IPv4 設定をここから書く**という宣言 |
| ├ `iface ens37` | 対象のインターフェース |
| ├ `inet` | IPv4 の設定（IPv6 なら `inet6`） |
| └ `static` | IP を自分で固定する方式（DHCP なら `dhcp`） |
| `address 10.0.0.1/24` | 付ける IP アドレスとネットワークの範囲 |
| `gateway 10.0.0.1` | （クライアント VM のみ）デフォルトゲートウェイ |

つまりこの設定は、手で打っていた次のコマンドを起動時に自動でやってくれる、ということです。

```bash
sudo ip link set ens37 up
sudo ip addr add 10.0.0.1/24 dev ens37
```

### `auto` と `allow-hotplug` の違い

| キーワード | いつ上がるか |
|---|---|
| `auto` | 起動時に上げる |
| `allow-hotplug` | NIC が認識された（差し込まれた）ときに上げる |

USB の LAN アダプタのように抜き差しされるものは `allow-hotplug` が向いています。VM の固定の NIC ならどちらでも動くので、わかりやすい `auto` で十分です。

### 書式のルール

| 項目 | ルール |
|---|---|
| **改行** | **必要**。1 行に 1 つの指示。`auto ens37 iface ens37 ...` と 1 行にまとめると、`auto` の後ろが全部インターフェース名として扱われて動かない |
| **行頭の字下げ** | **なくても動く**（見やすさのため）。`iface` 行の後に続く、`auto` や `iface` で始まらない行は、その `iface` の設定として扱われる。空白でもタブでも OK |
| **単語の間の空白** | 1 つ以上あれば OK |
| **空行** | 自由に入れてよい |
| **コメント** | `#` から行末まで |
| ⚠️ **全角スペース** | **エラーになる**。日本語入力のまま空白を打たないこと |

### 書いた後の文法チェック

```bash
sudo ifup --no-act ens37
```

`--no-act`（`-n`）は「実際には何もせず、何をするかだけ表示する」オプションです。エラーが出なければ、少なくとも書き方の誤りはありません。**再起動の前に必ずやっておく**と安心です。
