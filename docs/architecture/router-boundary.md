# Router Boundary

自作ルーターでは、どこまで自分で作るかによって「自作度」が変わる。

今回の方針は、**Linux Kernel以下は既存のものを利用し、ルーターとしての制御部分を理解・実装する**ことである。

ルーティングやNAT、ファイアウォールの**仕組みそのもの**は、Linuxカーネルが持っている。このリポジトリで作るのは、カーネルに「こう設定して」と頼む**コントローラー**（`routerd`）である。

```text
┌──────────────────────────────┐
│       自作する領域           │
│                              │
│ Router Controller（routerd） │
│ Routing Management           │
│ Firewall / NAT Management    │
│ Monitoring                   │
│ CLI / API                    │
│ DHCP / DNS（必要に応じて）   │
├──────────────────────────────┤
│  Linux標準のツール・窓口     │
│                              │
│ iproute2（ip コマンド）      │
│ nftables（nft コマンド）     │
│ Netlink（カーネルへの窓口）  │
├──────────────────────────────┤
│        Linux Kernel          │
│                              │
│ Routing Table                │
│ Netfilter / conntrack        │
│ TCP/IP Stack                 │
│ NIC Driver                   │
├──────────────────────────────┤
│          Hardware            │
└──────────────────────────────┘
```

---

## 作るもの・作らないもの

### 作らないもの：カーネルの仕組み

| 機能 | 仕組み（カーネルが持っている） |
|---|---|
| ルーティング | ルーティングテーブル、ロンゲストマッチで行を選ぶ、選んだ行に従ってパケットを送り出す |
| IP Forwarding | 自分宛てではないパケットを転送する |
| NAT | Netfilter がパケットの IP を書き換える。conntrack が書き換えを記録し、帰りを自動で書き戻す |
| ファイアウォール | Netfilter がルールに従ってパケットを通す・捨てる |

これらを作り直すことはしない。パケットを実際に運ぶ・書き換える・止めるのは、すべてカーネルである。

### 作るもの：カーネルを操作するコントローラー

| 仕事 | 例 | `routerd` の項目 |
|---|---|---|
| 設定する | ルーティングテーブルに行を足す・消す。nftables のルールを入れる | Routing / Firewall / NAT Management |
| 状態を見る | NIC が上がっているか、どの通信が通っているか | Traffic Monitoring |
| 人とやりとりする | 「設定を変えて」「状態を見せて」を受け付ける | CLI / API |

どれも、カーネルの機能を作り直すのではなく、**カーネルの機能をまとめて扱いやすくする**部分である。

---

## 手作業と routerd の関係

`ip route` や `nft` コマンドも、中では Netlink という窓口でカーネルに頼んでいる。`routerd` は、同じ窓口を自分のプログラムから使う。

```text
 Phase 3〜4：人     ──→ ip route add ... / nft add ... ──┐
                                                          ├──→ Netlink ──→ Linux Kernel
 Phase 6：  routerd ──→ Netlink / nft                   ──┘                     │
            ↑                                                                   ↓
         ここを作る                                         ルーティング・NAT・ファイアウォールを実際に動かす

 Phase 5：  configs/ ──Ansible──→ /etc/nftables.conf などの設定ファイル
            ↑
         ここを書く
```

| Phase | 誰がカーネルを操作するか | 何を作るか |
|---|---|---|
| Phase 3〜4 | 人（`ip`、`nft` コマンド） | ルールや設定を手で入れ、何が起きるかを確かめる |
| Phase 5 | Ansible（設定ファイルを配る） | `configs/` の設定ファイルと Playbook |
| Phase 6 | `routerd` | カーネルを操作するコントローラー |

---

## 手作業から始める理由

`routerd` の仕様は、「カーネルのどの機能を、どう設定すれば何が起きるか」で決まる。

そのため、Phase 3〜4 で手作業で確かめたことが、そのまま `routerd` に何をさせるかの仕様になる。例えば、

- labs/03 で `ip route add` / `ip route del` で行を足し消しした → Routing Management がやること
- labs/04 で nftables にマスカレードのルールを入れる → NAT Management がやること

手作業を飛ばして `routerd` を作ると、カーネルが何をしているかを知らないまま、操作だけを書くことになる。

---

## 市販のルーターとの関係

市販の家庭用ルーターの多くも、同じ作りである。中身は Linux カーネルで、その上にメーカーが作った管理ソフトと設定画面が載っている。

```text
 市販のルーター：  メーカーの管理ソフト・設定画面  →  Linux カーネル
 このリポジトリ：  routerd（CLI / API）           →  Linux カーネル
```

専用ルーターにたとえると、パケットを運ぶカーネルが ASIC に当たる部分で、`routerd` は設定を決める CPU 側の管理ソフトに近い（[専用ルーターはなぜ仕事を 2 つに分けるのか](../questions/2026-10-24-router-cpu-vs-asic.md)）。

---

## 関連

- [src/routerd](../../src/routerd/README.md)：`routerd` のイメージと開発思想
- [Overview](overview.md)：全体構成
- [007 - Ansible を使う](../decisions/007-use-ansible.md)：手作業の後に自動化する理由
