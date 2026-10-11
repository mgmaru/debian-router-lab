# DHCP サーバーはどこで動いているのか：VMware・Debian のサービス・実機

調査日：2026-10-12

関連：

- [DHCP サーバーはどう作るのか：仕組みと実装方針](2026-10-12-how-to-implement-dhcp-server.md)（質問12）
- [DHCP はルーターの機能か、DHCP サーバーはどこに・いくつ置くのか](2026-10-11-dhcp-role-and-placement.md)（質問9〜11）
- [DHCP サーバーのアドレスはどこで分かるのか](2026-10-11-where-to-find-dhcp-server-address.md)（質問8）
- [`192.168.22.2` は何か：VMware NAT とデフォルトゲートウェイ](2026-10-11-what-is-vmware-nat-gateway.md)（質問7）
- [docs/concepts/persistent-config.md（一時的な設定と永続的な設定）](../concepts/persistent-config.md)
- [labs/06-dhcp](../../labs/06-dhcp/README.md)、[deploy/README.md（VM から実機への移植）](../../deploy/README.md)

## 疑問

> **質問13（追加質問）**：Debian における DHCP サーバーは、何かのサービスで動いているのか？

> **質問14（追加質問）**：今は VMware に搭載されている DHCP サーバーを使っている、という認識でよいか？ この DHCP サーバーを置く設定はどこで行うのか？ サーバーを起動したり、逆にオフにしたりできるか？

> **質問15（追加質問）**：VMware の DHCP サーバーを使っているなら、将来、実機にプログラムを入れたときには、DHCP サーバーの機能が無いのではないか（VMware ではないので）。そうなると、Debian に DHCP サーバーのサービスを入れるか、自分で実装したプログラムを載せることになるのか？

## 結論

| 質問 | 答え |
|---|---|
| Debian の DHCP サーバーはサービスで動いている？ | **今の Debian には、DHCP サーバーはいない。** 動いているのは DHCP **クライアント**（dhcpcd）だけ。DHCP サーバーを入れると、**systemd のサービス**として動く |
| VMware の DHCP サーバーを使っている？ | **半分正解。** 使っているのは **Router VM の WAN 側（VMnet8）だけ**。LAN 側（VMnet2）には DHCP サーバーがいない |
| どこで設定する？ | Windows の VMware Workstation の「**仮想ネットワークエディタ**」。VMnet ごとに、DHCP サーバーを動かすかどうかを決める |
| 起動・停止できる？ | **できる。** VMnet ごとにチェックを外すか、Windows のサービス「VMware DHCP Service」を止める。ただし、**今 VMnet8 の DHCP を止めると、Router VM に SSH できなくなる** |
| 実機では DHCP サーバーが無くなる？ | VMware の DHCP は、実機でいえば **ISP（回線の会社）の役**。WAN 側は ISP が配ってくれる。**LAN 側の DHCP サーバーは、VM でももともと自分で用意するもの**で、実機でも同じ |
| サービスを入れるか、自作するか？ | **その通り。** 既存のサービス（dnsmasq や Kea）を入れるか、自作するか、その中間（既存のサービスを自作ソフトから設定する）の 3 通り。VM で作ったものは、そのまま実機に持っていける |

---

## 全体像：このラボで DHCP に関わっているもの

```text
Windows
 ├─ VMware DHCP Service（192.168.22.254）── VMnet8 に配る   ← DHCP サーバー（今あるのはこれだけ）
 │
 ├─ Router VM（Debian）
 │    ├─ ens33：dhcpcd が VMnet8 から受け取る               ← DHCP クライアント
 │    └─ ens37：静的 IP。まだ誰にも配っていない             ← labs/06 で DHCP サーバーになる
 │
 └─ Client VM（Debian）
      └─ ens33：静的 IP。DHCP を使っていない
```

| 立場 | 何が | どこで動いているか |
|---|---|---|
| DHCP **サーバー**（配る側） | VMware DHCP Service | **Windows** の中 |
| DHCP **クライアント**（受け取る側） | dhcpcd | **Router VM**（Debian）の中 |
| まだいないもの | LAN 側（VMnet2）の DHCP サーバー | labs/06 で **Router VM** に置く |

「DHCP サーバー」と「DHCP クライアント」を分けて考えるのがポイントである。

---

## 質問13：Debian の DHCP サーバーは、何かのサービスで動いているのか

### 今の Debian には、DHCP サーバーはいない

| VM / NIC | 動いているもの | 役割 |
|---|---|---|
| Router VM の ens33（WAN） | **dhcpcd** | DHCP **クライアント**。VMware の DHCP サーバーから設定を**もらう**側 |
| Router VM の ens37（LAN） | なし | 静的 IP。まだ誰にも配っていない |
| Client VM の ens33 | なし | 静的 IP |

### 動いている dhcpcd（クライアント）は、どう起動しているか

dhcpcd は、専用のサービスとして単独で起動しているのではなく、**ifupdown が NIC を起動するときに、一緒に起動している**と考えられる（推測を含む。下の方法で確かめられる）。

```text
起動時
  │
  ▼
ifupdown が /etc/network/interfaces を読む
  │
  │ iface ens33 inet dhcp を見つける
  ▼
ifup が dhcpcd を起動する
  │
  ▼
dhcpcd が VMware の DHCP サーバーから設定をもらう
```

`/etc/network/interfaces` の書き方によって、ifup を実行するサービスが変わる。

| 書き方 | ifup を実行するサービス | Router VM では |
|---|---|---|
| `auto ens37` | `networking.service`（起動時にまとめて実行） | ens37（LAN、静的 IP） |
| `allow-hotplug ens33` | `ifup@ens33.service`（NIC が見つかったときに実行） | ens33（WAN、DHCP） |

そのため、dhcpcd は `ifup@ens33.service` の一部として動いているはずである。

**確かめる方法（Router VM）：**

| コマンド | 分かること |
|---|---|
| `pgrep -a dhcpcd` | dhcpcd が動いているか、どんなオプションで起動したか |
| `systemctl status $(pgrep -o dhcpcd)` | その dhcpcd が、どのサービスの一部として動いているか（`ifup@ens33.service` の予想） |
| `sudo ss -ulpn` | UDP で待ち受けているプログラムの一覧。DHCP サーバーのポート `:67` は**出ないはず**（サーバーがいないから） |

### DHCP サーバーを入れたら、systemd のサービスとして動く

labs/06 で DHCP サーバーのソフトをパッケージで入れると、**systemd のサービス**として登録される。

| ソフト | パッケージ | サービス名 | 設定ファイル | リースの保存先 |
|---|---|---|---|---|
| **dnsmasq** | `dnsmasq` | `dnsmasq.service` | `/etc/dnsmasq.conf`、`/etc/dnsmasq.d/` | `/var/lib/misc/dnsmasq.leases` |
| **Kea** | `kea-dhcp4-server` | `kea-dhcp4-server.service` | `/etc/kea/kea-dhcp4.conf` | `/var/lib/kea/kea-leases4.csv` |

> ⚠️ `dnsmasq-base` というパッケージもある。こちらはプログラムだけで、サービスは付いていない。NetworkManager などが、プログラムとして内部で使うためのものである。サービスとして使うなら `dnsmasq` を入れる。

サービスになると、操作はほかのサービスと同じである。

| やりたいこと | コマンド（dnsmasq の例） |
|---|---|
| 状態を見る | `systemctl status dnsmasq` |
| 起動・停止・再起動 | `sudo systemctl start dnsmasq` / `stop` / `restart` |
| 起動時に自動で動かす（永続化） | `sudo systemctl enable dnsmasq` |
| ログを見る | `journalctl -u dnsmasq` |
| 67 番で待ち受けているか | `sudo ss -ulpn` に `:67` と `dnsmasq` が出る |

`systemctl enable` は、[persistent-config.md](../concepts/persistent-config.md) の「起動時に読まれる場所に書く」の一つである。DHCP サーバーも、`enable` しておかないと、再起動後に動かない。

### 自作するときも、systemd のサービスにする

自作の DHCP サーバー（や routerd）も、**systemd のサービスとして動かす**のが自然である。サービスの設定ファイル（ユニットファイル）を書くと、次のことを systemd に任せられる。

| systemd に任せられること | 自作すると大変なこと |
|---|---|
| 起動時に自動で起動する | 起動の仕組みを自分で用意する |
| 落ちたときに自動で再起動する | 監視の仕組みを自分で作る |
| ログを管理する（`journalctl`） | ログファイルの書き出しや整理 |
| 必要な権限だけを与えて、root 以外のユーザーで動かす | 権限を落とす処理（[質問12](2026-10-12-how-to-implement-dhcp-server.md#違い-4特別な権限が要る)） |

リポジトリには、すでに `deploy/systemd/` というフォルダがある（今は空）。自作のサービスのユニットファイルを置く場所になる。

---

## 質問14：VMware の DHCP サーバー

### 使っているのは、WAN 側（VMnet8）だけ

| ネットワーク | DHCP サーバー | 使っている機器 |
|---|---|---|
| VMnet8（Router VM の WAN 側） | **VMware の DHCP サーバー**（`.254`） | Router VM の ens33 |
| VMnet2（Router VM の LAN 側） | **無し** | 誰も使っていない（Client VM は静的 IP） |

### 設定する場所：仮想ネットワークエディタ

Windows の VMware Workstation で設定する。

1. メニューの「編集」→「**仮想ネットワークエディタ**」を開く
2. 右下の「設定の変更」を押す（管理者権限が必要）
3. 一覧から **VMnet8** を選ぶ

| 項目 | 意味 |
|---|---|
| 「ローカル DHCP サービスを使用して IP アドレスを VM に配布する」 | **このネットワークで DHCP サーバーを動かすかどうか**のチェック |
| 「DHCP 設定」ボタン | 配る範囲（開始と終了の IP）、リース時間 |
| 「NAT 設定」ボタン | ゲートウェイ IP（`192.168.22.2`） |
| サブネット IP / サブネットマスク | `192.168.22.0` / `255.255.255.0` |

**このチェックは、VMnet ごとにある。** VMnet2 を選ぶと、チェックが外れているはずである。labs/01 で Client VM に `169.254.x.x` が付いていたのは、そのためである。

> 💡 画面での設定は、裏では Windows の `C:\ProgramData\VMware\vmnetdhcp.conf` というファイルに保存されている（未確認）。中身は、ISC の DHCP サーバーの設定ファイルとよく似た書き方になっている。

### 起動・停止のしかた

| 方法 | 影響の範囲 |
|---|---|
| 仮想ネットワークエディタで、VMnet ごとのチェックを外す | **そのネットワークだけ**止まる |
| Windows の `services.msc` で「VMware DHCP Service」を停止する | **すべての VMnet** で止まる |

### VMnet8 の DHCP を止めると、どうなるか

Router VM の ens33 は、DHCP で IP をもらっている（リースは 30 分。[質問8](2026-10-11-where-to-find-dhcp-server-address.md#出力の読み方)）。

```text
DHCP サーバーを止める
  │ すぐには何も起きない（今のリースは有効なまま）
  ▼
延長の時刻（15 分ごと）：延長を申し込むが、返事が無い
  ▼
再取得の時刻：全員に呼びかけるが、返事が無い
  ▼
リースの期限切れ：ens33 の IP が無くなる
  ▼
Windows から Router VM に SSH できなくなる
```

再起動した場合は、その時点で IP をもらえなくなる。

> ⚠️ **今、VMnet8 の DHCP を止めるのはやめておく。** 試すなら、VMware のコンソールから操作できる状態にしてからにする。

> 💡 逆に、**VMnet2 の DHCP はオフのままにしておく**のが大事である。labs/06 で Router VM が DHCP サーバーになったとき、VMnet2 の VMware DHCP がオンだと、1 つのネットワークに DHCP サーバーが 2 台いる状態になる（[質問11](2026-10-11-dhcp-role-and-placement.md#質問11理解の確認とdhcp-サーバーの台数)）。

---

## 質問15：実機では、DHCP サーバーが無くなるのか

### VMware の DHCP は、実機では「ISP」の役

VMware の DHCP サーバーが配っている相手は、**Router VM の WAN 側**である。実機でいえば、**ISP（契約している回線の会社）**や、上流にある家庭用ルーターの役割に当たる。

**VM の場合（今）**

```text
インターネット
   │
VMware NAT ＋ VMware DHCP（Windows の中）   ← 上流の役
   │ VMnet8
Router VM の ens33：DHCP で受け取る
Router VM の ens37：labs/06 で配る          ← 自分のルーター
   │ VMnet2
Client VM
```

**実機の場合（将来）**

```text
インターネット
   │
ISP（または上流の家庭用ルーター）           ← 上流の役
   │ 光回線など
実機の WAN 側の NIC：DHCP で受け取る
実機の LAN 側の NIC：同じように配る         ← 自分のルーター
   │ LAN ケーブル・スイッチ
PC・スマホ
```

| 役割 | VM（今） | 実機（将来） | 誰が用意するか |
|---|---|---|---|
| WAN 側に**配る** DHCP サーバー | VMware DHCP | ISP、または上流のルーター | **外の誰か**。自分では作らない |
| WAN 側で**受け取る** DHCP クライアント | Router VM の dhcpcd | 実機 Debian の dhcpcd | Debian に入っているので、同じ |
| **LAN 側に配る** DHCP サーバー | **まだ無い**（labs/06 で作る） | **自分のルーター** | **自分** |

つまり、

- **VMware が無くなって困るのは WAN 側**だが、そこは **ISP が代わりに配ってくれる**
- **LAN 側の DHCP サーバーは、VM でも、もともと VMware の仕事ではない**。自分のルーターで用意する必要があり、それは実機でも同じ

> 💡 実機の WAN 側は、契約している回線によっては、DHCP ではなく **PPPoE** や **IPoE** という方式で設定を受け取る。日本の光回線では、どちらもよく使われる。その場合に変わるのは、WAN 側の「受け取る」仕組みだけである。LAN 側の DHCP サーバーの話は同じである。

### LAN 側の DHCP サーバーを、どう用意するか

ご想像のとおり、Debian にサービスを入れるか、自作のプログラムを載せるかである。もう 1 つ、その中間の方法もある。

| 方法 | 内容 | 長所 | 短所 | 向いている段階 |
|---|---|---|---|---|
| **A. 既存のサービスを入れる** | `apt` で dnsmasq や Kea を入れ、`systemctl enable` する | すぐ動く。実績があり安定している | 中身はブラックボックスのまま | まず動かして、仕組みを観察する（labs/06 の最初） |
| **B. 自作のプログラムを載せる** | [質問12](2026-10-12-how-to-implement-dhcp-server.md) の方針で作り、systemd のサービスにする | 仕組みを深く理解できる。自由に作れる | 作るのも、丈夫にするのも大変 | 仕組みを理解したい、自作の度合いを上げたい |
| **C. 既存のサービスを、自作ソフトから設定する** | routerd が dnsmasq の設定ファイルを書き出し、再起動する | DHCP 本体は安定したものを使いつつ、制御は自分で作れる | 設定ファイルの形式に合わせる必要がある | 自作ルーターの「制御部分」に集中したい（OpenWrt に近い考え方） |

labs/06 の方針（「最初は既存ソフトを使い、仕組みを理解した後に自作を検討する」）に沿うなら、**A から始めて、B か C に進む**流れになる。

```text
A. dnsmasq を入れて動かす（仕組みを観察する）
        │
        ├──▶ B. 自分で DHCP サーバーを作る（仕組みを深く理解したい）
        │
        └──▶ C. routerd から dnsmasq を設定する（制御部分を作りたい）
```

C の考え方は、[routerd](../../src/routerd/README.md) がカーネルの機能（ルーティング・NAT・ファイアウォール）を Netlink や nftables で操作するのと同じである。「本体は既存のものを使い、それを操作する部分を自作する」という形である。

### VM で作ったものは、そのまま実機に持っていける

VM でも実機でも、LAN 側の DHCP サーバーは**自分の Debian の上で**動く。そのため、VM で作ったものは実機でもほぼそのまま使える（[deploy/README.md](../../deploy/README.md) の「VM と実機で基本構造は変わらない」の通り）。

| 項目 | VM → 実機で変わるか |
|---|---|
| DHCP サーバーのソフト（dnsmasq、Kea、自作） | **変わらない** |
| systemd のサービスの設定 | **変わらない** |
| 配る範囲・ゲートウェイ（`10.0.0.0/24`、`10.0.0.1`） | **同じにできる** |
| 待ち受ける NIC の名前 | **変わる**（ens37 → 実機の LAN 側の NIC の名前） |
| WAN 側の受け取り方 | **回線しだい**（DHCP、PPPoE、IPoE） |

NIC の名前は変わるので、**自作するときは NIC の名前をプログラムに直接書かず、設定で変えられるようにしておく**とよい。

> 💡 実機の NIC の名前は、[質問4](2026-10-11-nic-name-and-multiple-ip-addresses.md#ens33-という名前の意味) で見た付け方で決まる。`deploy/README.md` では実機を `eth0` / `eth1` と書いているが、Debian の実機では `enp1s0` のような名前になるのが普通である。

---

## まだ試していないこと

| どこで | 操作 | 予想される結果 | 確かめられること |
|---|---|---|---|
| Router VM | `pgrep -a dhcpcd` | dhcpcd が ens33 用に動いている | Debian で動いているのは DHCP クライアント |
| Router VM | `systemctl status $(pgrep -o dhcpcd)` | `ifup@ens33.service` の一部として表示される | dhcpcd は ifupdown から起動されている |
| Router VM | `sudo ss -ulpn` | `:67` の行が無い | Debian に DHCP サーバーはいない |
| Windows | 仮想ネットワークエディタで VMnet8 と VMnet2 を見比べる | VMnet8 は DHCP のチェックあり、VMnet2 はなし | DHCP サーバーは VMnet ごとにオン・オフが決まっている |
| Windows | VMnet8 の「DHCP 設定」を開く | 配る範囲に `192.168.22.129` が含まれている | Router VM の IP は、この範囲から配られた |
| Windows | `services.msc` を開く | 「VMware DHCP Service」が実行中 | VMware の DHCP サーバーは Windows のサービス |

---

## まとめ

- このラボで今 DHCP **サーバー**として動いているのは、**Windows の VMware DHCP Service だけ**。Debian で動いている dhcpcd は DHCP **クライアント**で、ifupdown が NIC を起動するときに一緒に起動している。
- Debian に DHCP サーバー（dnsmasq や Kea）を入れると、**systemd のサービス**として動く。`systemctl enable` で永続化する。自作するときも、systemd のサービスにするのが自然。
- VMware の DHCP サーバーは、**仮想ネットワークエディタで VMnet ごとに**オン・オフできる。ただし、**VMnet8 を止めると Router VM に SSH できなくなる**。VMnet2 はオフのままにしておく。
- VMware の DHCP は、実機でいえば **ISP の役**。WAN 側は ISP が配ってくれる。**LAN 側の DHCP サーバーは、VM でも実機でも自分で用意する**。
- 用意の仕方は、**A. 既存のサービス**、**B. 自作**、**C. 既存のサービスを自作ソフトから設定する**の 3 通り。**A から始めて B か C に進む**。VM で作ったものは、NIC の名前以外はそのまま実機に持っていける。
