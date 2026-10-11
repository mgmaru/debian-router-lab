# `192.168.22.2` は何か：VMware NAT とデフォルトゲートウェイ

調査日：2026-10-11

関連：

- [docs/concepts/neighbor.md（このラボで見える相手）](../concepts/neighbor.md#このラボで見える相手)
- [`ip addr` にデフォルトゲートウェイは表示されないのか](2026-10-11-ip-addr-and-default-gateway.md)
- [docs/concepts/routing.md（ルーター VM に `gateway` を書かなかった理由）](../concepts/routing.md#ルーター-vm-に-gateway-を書かなかった理由)
- [docs/concepts/dhcp.md](../concepts/dhcp.md)
- [DHCP サーバーのアドレスはどこで分かるのか](2026-10-11-where-to-find-dhcp-server-address.md)（追加質問：質問8）

## 疑問

> **質問7（追加質問）**：`192.168.22.1` が Windows 側の IP なのは、`ipconfig` で分かった。では、`192.168.22.2` は何か？
> デフォルトゲートウェイは Router VM の `ip addr` では出てこないが、これは DHCP で割り振られているからか？
> また、DHCP は Router VM の NIC に設定されているはずで、相手ではない気がする。デフォルトゲートウェイは相手として見るのか？

### 確認できていること

Windows で `ipconfig` を実行した結果（抜粋）：

```text
イーサネット アダプター VMware Network Adapter VMnet8:

   接続固有の DNS サフィックス . . . . .:
   リンクローカル IPv6 アドレス. . . . .: fe80::dab:d883:a1bb:ce8d%43
   IPv4 アドレス . . . . . . . . . . . .: 192.168.22.1
   サブネット マスク . . . . . . . . . .: 255.255.255.0
   デフォルト ゲートウェイ . . . . . . .:
```

## 結論

| 質問 | 答え |
|---|---|
| `192.168.22.2` は何？ | **VMware が Windows の中で動かしている仮想ルーター（VMware NAT）**。Windows 自身（`.1`）とは別の相手で、Router VM のデフォルトゲートウェイ |
| `ip addr` に出ないのは DHCP だから？ | **いいえ。** DHCP でも静的設定でも、`ip addr` にはデフォルトゲートウェイは出ない。`ip route` で見る |
| DHCP は自分の NIC の設定では？ | 自分の NIC にあるのは **DHCP クライアント**（設定をもらう側）。設定を配っているのは **DHCP サーバー**（`.254`）という別の相手 |
| デフォルトゲートウェイは相手か？ | **相手である。** 自分の側にあるのは「誰に渡すか」というメモ（ルーティングテーブルの行）だけ |

---

## `192.168.22.2` は何か

### VMnet8 にいる相手

VMnet8（`192.168.22.0/24`）には、Router VM のほかに、VMware 側の相手が 3 つつながっている。

| IP | 正体 | 役割 |
|---|---|---|
| `192.168.22.1` | Windows の VMnet8 アダプタ（Windows 自身） | Windows が VMnet8 に参加するための窓口。SSH の接続元 |
| `192.168.22.2` | **VMware NAT**（Windows 上で動く「VMware NAT Service」） | VM の通信を、Windows の物理 NIC 経由でインターネットへ中継する |
| `192.168.22.254` | VMware DHCP（Windows 上で動く「VMware DHCP Service」） | VM に IP アドレスやゲートウェイを配る |
| `192.168.22.129` | Router VM の ens33 | 自分 |

`.1`、`.2`、`.254` は、どれも物理的には同じ Windows PC の中で動いている。それでもネットワーク上では、**別々の IP と MAC を持つ、別の相手**である（[neighbor.md](../concepts/neighbor.md#このラボで見える相手) の `ip neigh` でも、3 つとも別の MAC で載っていた）。

### Router VM からインターネットへの道

```text
Router VM（ens33：192.168.22.129）
    │ ① default via 192.168.22.2 なので、.2 の MAC 宛てに渡す
    ▼
VMware NAT（192.168.22.2）
    │ ② 送信元を Windows の IP に書き換えて中継する
    ▼
Windows の物理 NIC（Wi-Fi など）
    │
    ▼
インターネット
```

**`.1`（Windows の VMnet8 アダプタ）は、この道の途中にいない。** `ipconfig` で VMnet8 アダプタのデフォルトゲートウェイが空欄なのも、そのためである。Windows 自身は、インターネットへ出るときに VMnet8 を使わず、物理 NIC から直接出ている。VMnet8 アダプタは、Windows と VM が話すための窓口にすぎない。

> 💡 VMware の仮想ネットワークエディタでは、VMnet8 に Windows のアダプタをつなぐかどうかを選べる。つながなくても、VM は NAT 経由でインターネットに出られる。ただし、Windows から VM に SSH できなくなる。

---

## `ip addr` に出ないのは DHCP だからか

**いいえ。DHCP は関係ない。** `ip addr` は NIC に付いたアドレスを見るコマンドなので、ゲートウェイの入り方に関係なく、デフォルトゲートウェイは出ない（[以前の疑問](2026-10-11-ip-addr-and-default-gateway.md)）。

| VM | ゲートウェイの入り方 | `ip addr` に出るか | `ip route` の行 |
|---|---|---|---|
| Router VM | DHCP でもらった | 出ない | `default via 192.168.22.2 dev ens33 proto dhcp ...` |
| Client VM | 設定ファイルに `gateway` で書いた | 出ない | `default via 10.0.0.1 dev ens33` |

DHCP が関係するのは、**どうやってこの行が入ったか**（`proto dhcp`）だけである。

---

## DHCP は自分の NIC の設定ではないのか

2 つのものが混ざっている。

| もの | どこにあるか | 役割 |
|---|---|---|
| **DHCP クライアント**（`iface ens33 inet dhcp`、dhcpcd） | **自分**（Router VM の ens33） | 設定を**もらう**側 |
| **DHCP サーバー**（`192.168.22.254`） | **相手** | 設定を**配る**側 |

`iface ens33 inet dhcp` は「この NIC は、設定を DHCP サーバーからもらう」という意味である。

そして、もらった設定の中に「ゲートウェイは `192.168.22.2`」という情報が入っている。`.2` は、DHCP サーバーとはさらに別の相手である。

```text
DHCP サーバー（.254）──「ゲートウェイは .2 です」──▶ Router VM（.129）
                                                        │
                                                        │ 外へのパケット
                                                        ▼
                                                   VMware NAT（.2）
```

> 💡 [routing.md](../concepts/routing.md#たとえ) の会社のたとえでいうと、新入社員（Router VM）が総務（DHCP サーバー）から「あなたの席は 129 番。社外宛ての書類は発送窓口（`.2`）へ」と教えてもらうイメージである。総務と発送窓口は別の人で、新入社員はどちらとも違う。

---

## デフォルトゲートウェイは相手として見るのか

**相手である。** 自分の側にあるのは、「誰に渡すか」というメモだけである。

| もの | 自分か相手か | どこで見えるか |
|---|---|---|
| `192.168.22.129`（自分の IP） | 自分 | `ip addr` |
| `default via 192.168.22.2`（ゲートウェイの設定） | 自分が持っているメモ | `ip route` |
| `192.168.22.2`（ゲートウェイの機器 ＝ VMware NAT） | **相手** | `ip neigh`（通信したことがあれば） |
| `192.168.22.254`（DHCP サーバー） | **相手** | `ip neigh`（通信したことがあれば） |

### Client VM と Router VM は同じ関係

| | Client VM | Router VM |
|---|---|---|
| 自分の IP | `10.0.0.10` | `192.168.22.129` |
| デフォルトゲートウェイ（相手） | `10.0.0.1`（Router VM の ens37） | `192.168.22.2`（VMware NAT） |
| `ip neigh` に載ったゲートウェイの MAC | `00:0c:29:73:2d:6b` | `00:50:56:f9:12:3d` |

**Router VM にとっての VMware NAT は、Client VM にとっての Router VM と同じ立場**である。

このラボでは、Router VM に IP フォワーディング（labs/02）や NAT（labs/04）を設定していく。それは、VMware NAT が Router VM にしていることを、今度は Router VM が Client VM にできるようにする、ということでもある。

---

## 検証：`dhcpcd -U` でゲートウェイと DHCP サーバーを見る（2026-10-11）

Router VM で `sudo dhcpcd -U ens33` を実行した結果（抜粋）：

```text
routers=192.168.22.2
domain_name_servers=192.168.22.2
dhcp_server_identifier=192.168.22.254
```

| 項目 | 値 | 分かること |
|---|---|---|
| `routers` | `192.168.22.2` | デフォルトゲートウェイは `.2`（VMware NAT） |
| `dhcp_server_identifier` | `192.168.22.254` | 設定を配った DHCP サーバーは `.254` |
| `domain_name_servers` | `192.168.22.2` | DNS サーバーも `.2`（VMware NAT） |

**ゲートウェイ（`.2`）と DHCP サーバー（`.254`）は別の相手**であることが確かめられた。全体の出力と読み方は [質問8](2026-10-11-where-to-find-dhcp-server-address.md#出力の読み方) を参照。

---

## まだ試していないこと

| どこで | 操作 | 予想される結果 | 確かめられること |
|---|---|---|---|
| Windows | VMware の仮想ネットワークエディタで、VMnet8 の「NAT 設定」を開く | ゲートウェイ IP が `192.168.22.2` | `.2` は VMware NAT |
| Windows | `services.msc` を開く | 「VMware NAT Service」と「VMware DHCP Service」が実行中 | NAT と DHCP は Windows の中で動いている |
| Windows | `arp -a` | VMnet8（`192.168.22.1`）の欄に、`192.168.22.129` と `00-0c-29-73-2d-61` が出る | Windows 側から見ると、Router VM が「相手」として載る |

---

## まとめ

- `192.168.22.2` は、VMware が Windows の中で動かしている**仮想ルーター（VMware NAT）**。Windows 自身（`.1`）とは別の相手で、Router VM のデフォルトゲートウェイ。
- `ip addr` にゲートウェイが出ないのは、**DHCP だからではない**。ゲートウェイはルーティングテーブルの 1 行なので、`ip route` で見る。
- 自分の NIC にあるのは **DHCP クライアント（もらう側）**。配っているのは **DHCP サーバー（`.254`）**、配られた中身が指しているのが **ゲートウェイ（`.2`）**。3 つとも別のもの。
- デフォルトゲートウェイは**相手**。自分の側にあるのは「誰に渡すか」というメモだけ。
- Router VM にとっての VMware NAT は、**Client VM にとっての Router VM と同じ立場**。
