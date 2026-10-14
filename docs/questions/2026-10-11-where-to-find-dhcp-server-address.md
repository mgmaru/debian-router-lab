# DHCP サーバーのアドレスはどこで分かるのか

調査日：2026-10-11

関連：

- [docs/concepts/dhcp.md（DHCP のやりとり、受け取った内容の確認）](../concepts/dhcp.md)
- [`192.168.22.2` は何か：VMware NAT とデフォルトゲートウェイ](2026-10-11-what-is-vmware-nat-gateway.md)（質問7）
- [`ip addr` にデフォルトゲートウェイは表示されないのか](2026-10-11-ip-addr-and-default-gateway.md)
- [labs/01-network-interface](../../labs/01-network-interface/README.md)
- [NIC 名（`ens33`）と、1 つの NIC に付ける IP アドレス（数・ネットワーク）](2026-10-11-nic-name-and-multiple-ip-addresses.md)（質問5：1 つの NIC に複数の IP を付ける）

## 疑問

> **質問8（追加質問）**：DHCP サーバーは `ip addr` でも出てこないのではないか？ DHCP サーバーのアドレスはどこで分かるのか？

> **質問30（追加質問）**：`sudo dhcpcd -U ens33` は、ens33 のインターフェースに対する DHCP クライアントの結果か？ ens33 には複数の IP を割り振れると思うが、今は `ip_address` が `192.168.22.129` になっている。ここに別のアドレスを割り振った場合、`ip_address` は複数表示されるのか？（2026-10-14 追記）

## 結論

| 質問 | 答え |
|---|---|
| `ip addr` に DHCP サーバーは出ない？ | **その通り、出ない。** `ip addr` は自分の NIC に付いたアドレスを見るコマンドで、DHCP サーバーは**相手**だから。`ip addr` で分かるのは、「DHCP でもらったアドレスであること（`dynamic`）」までである |
| どこで分かる？ | **DHCP クライアントが受け取った情報**で分かる。Router VM なら `sudo dhcpcd -U ens33` の `dhcp_server_identifier` に出る（このラボでは `192.168.22.254` だった） |
| `dhcpcd -U` に、手で付けた IP も出る？ | **出ない。** `dhcpcd -U` に出るのは、ens33 の dhcpcd が DHCP で受け取った内容だけ。ens33 に付いているすべての IP は `ip addr show ens33` で見る |

---

## なぜ `ip` コマンドには出てこないのか

DHCP サーバーは、**設定を配り終えたら、普段の通信では使わない相手**だからである。

| もの | 普段の通信で使うか | どこに残っているか |
|---|---|---|
| 自分の IP（`192.168.22.129`） | 使う（送信元） | `ip addr` |
| ゲートウェイ（`192.168.22.2`） | 使う（外への渡し先） | `ip route` |
| DHCP サーバー（`192.168.22.254`） | **使わない**（リースの更新のときだけ話す） | DHCP クライアント（dhcpcd）が受け取った情報の中 |

カーネルが持っているのは、パケットを送るのに必要な情報（アドレス・経路・近隣）だけである。DHCP サーバーのアドレスはパケットを送るのに必要ないので、カーネルではなく**DHCP クライアント（dhcpcd）が覚えている**。

### コマンドごとに分かること

| コマンド | DHCP サーバーのアドレスは出るか | 理由 |
|---|---|---|
| `ip addr` | 出ない | 自分のアドレスだけ。`dynamic` で「DHCP でもらった」ことまでは分かる |
| `ip route` | 出ない | 経路だけ。ゲートウェイ（`.2`）は出るが、DHCP サーバー（`.254`）とは別 |
| `ip neigh` | `192.168.22.254` が載ることはある | ただし「最近通信した相手」というだけで、それが DHCP サーバーだとは分からない |
| `sudo dhcpcd -U ens33` | **出る** | DHCP クライアントが受け取った情報そのもの |
| `journalctl`（ログ） | 出る（予想） | dhcpcd が、どのサーバーから受け取ったかをログに残す |
| `tcpdump` | 出る | DHCP のやりとりそのものを見る |

> 💡 家庭のルーターでは、DHCP サーバーとデフォルトゲートウェイが同じ機器（たとえば `192.168.1.1`）であることが多い。そのため「ゲートウェイ ＝ DHCP サーバー」と思い込みやすい。このラボの VMnet8 では、ゲートウェイ（`.2`）と DHCP サーバー（`.254`）が別である（[質問7](2026-10-11-what-is-vmware-nat-gateway.md)）。

---

## 確認方法 1：`dhcpcd -U`（いちばん簡単）

```bash
sudo dhcpcd -U ens33
```

DHCP で受け取った情報が `名前=値` の形で一覧表示される。

### Router VM での実行結果（2026-10-11）

```text
kajih@debian-router:/etc/network$ sudo dhcpcd -U ens33
reason=CARRIER
interface=ens33
protocol=link
if_configured=true
ifcarrier=up
ifmetric=1002
ifwireless=0
ifflags=69699
ifmtu=1500

reason=RENEW
interface=ens33
protocol=dhcp
ip_address=192.168.22.129
subnet_cidr=24
network_number=192.168.22.0
subnet_mask=255.255.255.0
routers=192.168.22.2
domain_name_servers=192.168.22.2
domain_name=localdomain
broadcast_address=192.168.22.255
dhcp_lease_time=1800
dhcp_message_type=5
dhcp_server_identifier=192.168.22.254
dhcp_renewal_time=900
dhcp_rebinding_time=1575
```

**`dhcp_server_identifier=192.168.22.254` と `routers=192.168.22.2` は別の値だった。** DHCP サーバーとゲートウェイは別の相手であることが確かめられた。

### 出力の読み方

出力は、空行で区切られた 2 つのかたまりになっている。

**かたまり 1：`reason=CARRIER`（NIC の状態）**

| 項目 | 意味 |
|---|---|
| `protocol=link` | IP より下の、NIC そのものの情報 |
| `ifcarrier=up` | ケーブルがつながっている（仮想的に） |
| `ifmetric=1002` | `ip route` に出ていた `metric 1002` と同じ値 |
| `ifwireless=0` | 無線ではない |
| `ifmtu=1500` | `ip link` の `mtu 1500` と同じ値 |
| `ifflags=69699` | NIC の状態を数字で表したもの。`ip link` の `<BROADCAST,MULTICAST,UP,LOWER_UP>` を数字にした値 |

**かたまり 2：`reason=RENEW`（DHCP で受け取った内容）**

`RENEW` は、最後の出来事がリースの延長だった、という意味である。

| 項目 | 値 | 意味 |
|---|---|---|
| `ip_address` | `192.168.22.129` | もらった自分の IP |
| `subnet_cidr` / `subnet_mask` | `24` / `255.255.255.0` | ネットワークの範囲（同じ意味の 2 通りの書き方） |
| `network_number` / `broadcast_address` | `192.168.22.0` / `192.168.22.255` | ネットワーク自身のアドレスと、全員宛てのアドレス |
| `routers` | `192.168.22.2` | **デフォルトゲートウェイ**（VMware NAT） |
| `domain_name_servers` | `192.168.22.2` | **DNS サーバー**。VMware NAT が DNS も担当している |
| `domain_name` | `localdomain` | ドメイン名 |
| `dhcp_server_identifier` | `192.168.22.254` | **この設定を配った DHCP サーバーのアドレス** |
| `dhcp_message_type` | `5` | 最後に受け取ったメッセージの種類。`5` は Ack（DORA の最後の「どうぞ」） |
| `dhcp_lease_time` | `1800` | リースの期限は 30 分 |
| `dhcp_renewal_time` | `900` | 15 分たったら、同じサーバーに延長を申し込む |
| `dhcp_rebinding_time` | `1575` | 26 分 15 秒たっても延長できなければ、全体に呼びかけて別のサーバーを探す |

> ⚠️ インターフェース名を間違えて `sudo dhcpcd -U ens3` と打つと、`dhcpcd is not running` と表示された。`dhcpcd -U` は、そのインターフェースを担当している dhcpcd に問い合わせるコマンドで、`ens3` を担当する dhcpcd はいないからである。DHCP が止まっているわけではない。

DHCP のやりとり（[DORA](../concepts/dhcp.md#dhcp-のやりとりdora)）で、サーバーは Offer や Ack の中に「自分のアドレス」を入れて送ってくる。`dhcp_server_identifier` は、その値である。

---

## 確認方法 2：ログを見る

dhcpcd は、どのサーバーからアドレスを受け取ったかをログに残す。

```bash
sudo journalctl -b | grep dhcpcd
```

次のような行が出るはずである（予想）。

```text
dhcpcd[...]: ens33: offered 192.168.22.129 from 192.168.22.254
dhcpcd[...]: ens33: leased 192.168.22.129 for 1800 seconds
```

`from 192.168.22.254` の部分が、DHCP サーバーのアドレスである。

`-b` は「今回の起動以降のログだけ」という意味である。

---

## 確認方法 3：`tcpdump` でやりとりを見る

DHCP の通信を直接見る方法である。DHCP はポート 67（サーバー側）と 68（クライアント側）を使う。

```bash
sudo tcpdump -i ens33 -n port 67 or port 68
```

dhcpcd は、リースの期限の半分くらいでサーバーに延長を申し込む。そのタイミングで、次のような行が見えるはずである（予想）。

```text
IP 192.168.22.129.68 > 192.168.22.254.67: BOOTP/DHCP, Request ...
IP 192.168.22.254.67 > 192.168.22.129.68: BOOTP/DHCP, Reply ...
```

`192.168.22.254.67`（ポート 67 側）が DHCP サーバーである。

> ⚠️ リースの期限が 30 分なら、延長は 15 分ごとくらいなので、しばらく待つ必要がある。ens33 は SSH にも使っている NIC なので、延長を待たずに DHCP をやり直すような操作はしないほうが安全である。`Ctrl+C` で止める。

---

## Windows の場合

Windows では、`ipconfig /all` に「DHCP サーバー」の行が出る。

```text
DHCP 有効 . . . . . . . . . . . . . . : はい
DHCP サーバー . . . . . . . . . . . . : 192.168.1.1
```

ただし、これが出るのは DHCP で設定をもらっているアダプタ（Wi-Fi など）だけである。VMware Network Adapter VMnet8 は Windows 側で固定の `192.168.22.1` なので、DHCP サーバーの行は出ない。

Linux は Windows と違い、`ip` コマンドが DHCP の情報を持っていないので、DHCP クライアント（dhcpcd）に聞く必要がある。

---

## 質問30：`dhcpcd -U` に、手で付けた IP も表示されるか

### 結論

| 質問 | 答え |
|---|---|
| `sudo dhcpcd -U ens33` は、ens33 の DHCP クライアントの結果？ | **その通り。** ens33 を担当している dhcpcd（DHCP クライアント）が、**DHCP サーバーから受け取った内容**（リース）を表示している |
| 別のアドレスを付けたら、`ip_address` が複数表示される？ | **されない。** `dhcpcd -U` に出るのは、DHCP でもらったアドレスだけである。`ip addr add` で手で付けたアドレスは DHCP と関係ないので、表示されない。ens33 に付いているアドレスを**すべて**見たいときは、`ip addr show ens33` を使う |

### `dhcpcd -U` と `ip addr` は、見ているものが違う

| | `sudo dhcpcd -U ens33` | `ip addr show ens33` |
|---|---|---|
| 何を表示するか | dhcpcd が **DHCP で受け取った内容** | ens33 に**今付いているすべての IP** |
| 誰に聞いているか | dhcpcd（DHCP クライアント） | カーネル |
| DHCP でもらった IP | 出る（`ip_address`） | 出る（`dynamic` が付く） |
| 手で付けた IP | **出ない** | 出る |
| ほかに分かること | ゲートウェイ、DNS、DHCP サーバー、リース時間 | MAC アドレス、状態、リースの残り時間（`valid_lft`） |

たとえるなら、`dhcpcd -U` は「**DHCP サーバーから受け取った貸出証**」を見るコマンドである。`ip addr` は「**今、胸に付けている名札を全部**」見るコマンドである。自分で作った名札（手で付けた IP）は、貸出証には載っていない。

### 1 回の DHCP でもらえる IP は 1 つ

DHCP サーバーは、1 つの貸し出し（リース）につき、IP アドレスを 1 つ配る。そのため、ens33 の dhcpcd が持っている `ip_address` は、いつも 1 つである。

### 実際に付けた場合（予想）

たとえば、ens33 に `192.168.22.50/24` を手で付けたとする。

```bash
sudo ip addr add 192.168.22.50/24 dev ens33
```

`ip addr show ens33` には、2 つ並ぶはずである。

```text
inet 192.168.22.129/24 brd 192.168.22.255 scope global dynamic noprefixroute ens33
   valid_lft 1520sec preferred_lft 1295sec
inet 192.168.22.50/24 scope global secondary ens33
   valid_lft forever preferred_lft forever
```

| 行 | 見分け方 |
|---|---|
| `192.168.22.129` | `dynamic` と、期限のある `valid_lft` → DHCP でもらったもの |
| `192.168.22.50` | `dynamic` が無く `valid_lft forever`。同じネットワークの 2 つ目なので `secondary`（[質問5](2026-10-11-nic-name-and-multiple-ip-addresses.md#追加方法一時的)） → 手で付けたもの |

一方、`sudo dhcpcd -U ens33` の `ip_address` は、`192.168.22.129` のままのはずである。

> ⚠️ 試すときは、VMware の DHCP が配る範囲（通常は `.128`〜`.254`）の外のアドレスを使う。範囲の中のアドレスを付けると、ほかの VM に配られて重複するおそれがある。ens33 は SSH に使っている NIC だが、アドレスを**足す**だけなら接続は切れない（DHCP のアドレスを消すと切れる）。

---

## まだ試していないこと

| コマンド（Router VM） | 予想される結果 | 確かめられること |
|---|---|---|
| `sudo journalctl -b \| grep dhcpcd` | `offered 192.168.22.129 from 192.168.22.254` のような行 | ログからも DHCP サーバーが分かる |
| `sudo tcpdump -i ens33 -n port 67 or port 68`（延長まで待つ） | `192.168.22.254.67` とのやりとり | DHCP の通信を直接見られる |
| `sudo ip addr add 192.168.22.50/24 dev ens33` → `ip addr show ens33` と `sudo dhcpcd -U ens33` を見比べる → `sudo ip addr del 192.168.22.50/24 dev ens33` で戻す | `ip addr` には 2 つ並ぶが、`dhcpcd -U` の `ip_address` は `192.168.22.129` だけ | `dhcpcd -U` に出るのは、DHCP で受け取った内容だけ（質問30） |

---

## まとめ

- DHCP サーバーは**相手**なので、`ip addr` には出ない。`ip addr` で分かるのは「DHCP でもらった（`dynamic`）」ことまで。
- DHCP サーバーのアドレスは、パケットを送るのに必要ないので、カーネルではなく **DHCP クライアント（dhcpcd）が覚えている**。
- いちばん簡単な確認方法は `sudo dhcpcd -U ens33` の **`dhcp_server_identifier`**。ログ（`journalctl`）や `tcpdump` でも分かる。
- このラボでは、**DHCP サーバー（`.254`）とゲートウェイ（`.2`）は別の相手**。家庭のルーターのように同じとは限らない。
- `dhcpcd -U` に出るのは、**DHCP で受け取った内容だけ**。手で付けた IP は出ないので、ens33 に付いているすべての IP は `ip addr show ens33` で見る。
