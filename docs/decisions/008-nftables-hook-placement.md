# 008. ファイアウォールは input と forward に、NAT は postrouting に置く

## Status

Accepted

## Context

Router VM のファイアウォールと NAT は、nftables で設定する。nftables の rule は chain に入れ、chain はカーネルの関所（Netfilter のフック）に置く。

IPv4 の関所は5つある（[netfilter-hooks.md](../concepts/netfilter-hooks.md)）。

| 関所 | 通るパケット | その時点でわかっていること |
|---|---|---|
| ① prerouting | 届いたパケットすべて | 入ってきた NIC。出口はまだ決まっていない |
| ② input | Router VM 宛てだけ | 自分宛てだと決まった |
| ③ forward | 通り抜けるものだけ | 入口と出口の両方 |
| ④ output | Router VM が作ったものだけ | 出口 |
| ⑤ postrouting | 出ていくパケットすべて | 出口が決まった |

カーネルの制限は、「NAT の chain（`type nat`）は forward に置けない」の1つだけである。filter の chain は5つのどこにでも置ける。そのため、同じ目的の rule を別の関所に置くこともできる。

| 目的 | 置ける関所の例 |
|---|---|
| WAN からのパケットを捨てる | ① prerouting（届いた直後に捨てる）、② input と ③ forward（行き先が決まってから捨てる） |
| Router VM が出す通信を制限する | ④ output |
| 外から内側のサーバーへ通す（ポート転送） | ① prerouting（DNAT） |

置く場所を決めておかないと、labs/05 で rule を書くとき、Phase 5 で Ansible に書き起こすとき、Phase 6 で `routerd` の Firewall / NAT Management を作るときに、書き方がぶれる。

## Decision

nftables の rule は、次の関所に置く。

| 置くもの | 関所 | chain の type | 状態 |
|---|---|---|---|
| Router VM を守るファイアウォール | ② input | `filter` | labs/05 で書く |
| LAN 側の機器を守るファイアウォール | ③ forward | `filter` | labs/05 で書く |
| 送信元の書き換え（マスカレード） | ⑤ postrouting | `nat` | labs/04 で設定済み（`oifname "ens33" masquerade`） |

次のものは置かない。

| 置かないもの | 関所 |
|---|---|
| Router VM が出す通信の制限 | ④ output（chain は残すが、rule は書かない） |
| 宛先の書き換え（DNAT、ポート転送） | ① prerouting |
| ファイアウォールの rule | ① prerouting・⑤ postrouting |

## Reasons

### ファイアウォールを input と forward に置く理由

- **守る対象と関所が1対1で対応する**。Router VM 宛てのパケットは input だけを、通り抜けるパケットは forward だけを通る。そのため、「Router VM を守る rule」と「LAN 側の機器を守る rule」を別々の chain に分けて書ける
- **forward では入口と出口の両方がわかる**。`iifname "ens37" oifname "ens33"`（LAN → WAN）のように、向きで rule を書ける。prerouting ではまだ出口が決まっていないので、向きで書けない
- **NAT を気にせずに書ける**。forward の時点では、行きの送信元はまだ書き換えられておらず（書き換えは postrouting）、帰りの宛先はもう書き戻されている（書き戻しは prerouting）。そのため、Client VM の本当のアドレス（`10.0.0.10`）と NIC の名前で書ける
- **conntrack の判定が済んでいる**。conntrack は prerouting で動くので、input と forward では `ct state established` / `new` で「返事」と「新しい接続」を区別できる
- Linux ルーターの一般的な書き方であり、nftables の公式 wiki の例や、ほかの資料と照らし合わせやすい

### NAT（マスカレード）を postrouting に置く理由

- 送信元を「出口の NIC の IP」に書き換えるので、**出口が決まってから**でないと書き換えられない。出口が決まっているのは postrouting だけである（output も出口はわかるが、Router VM が作ったパケットしか通らない）
- 帰りの書き戻しは、conntrack の記録を使ってカーネルが自動で行う。そのため、帰り用の rule は書かない

### output を制限しない理由

- Router VM が自分から出す通信は、`apt`、DNS の問い合わせ、SSH の返事などで、守る対象ではない
- 家庭用ルーターでも、ルーター自身が出す通信はほとんど制限しない
- output を制限すると、`apt update` が通らないなど、ファイアウォールの学習とは関係のない原因で困ることが増える

### DNAT（ポート転送）を使わない理由

- LAN 側に、外から接続させたいサーバーが無い
- labs/05 の目的は「外から始まった新しい接続を止める」ことであり、外から内側へ通す設定はその逆になる

### ファイアウォールの rule を prerouting に置かない理由

- prerouting に置くと、Router VM 宛てのパケットと、通り抜けるパケットが同じ chain に混ざる
- 出口がまだ決まっていないので、向き（LAN → WAN など）で書けない
- 届いた直後に捨てると負荷を減らせるが、このラボの通信量ではその必要がない

## Consequences

- `/etc/nftables.conf`（[configs/nftables/nftables.conf](../../configs/nftables/nftables.conf)）の構成は、次のようになる
  - `table inet filter`：`input`・`forward`・`output` の3つの chain。rule を書くのは `input` と `forward` だけ
  - `table ip nat`：`postrouting` の chain だけ
- labs/05 では、`table inet filter` の `input` と `forward` に rule を書き足す
- Phase 5 の Ansible、Phase 6 の `routerd` も、この置き場所に合わせて rule を生成する

## 見直す条件

| 起きたこと | 見直す内容 |
|---|---|
| LAN 側にサーバーを置き、外から接続させたくなった | ① prerouting に DNAT の chain を足す。forward にも、その通信を許可する rule が要る |
| Router VM が出す通信を制限したくなった | ④ output の chain に rule を書く |
| 実機へ移植し、通信量が多くて負荷が問題になった | ① prerouting や netdev の ingress で、早めに捨てることを検討する |

## Notes

この ADR が決めるのは「どの関所に置くか」だけである。input と forward を `policy drop` にするか、どのポート（SSH・DHCP・DNS など）を許可するかは、labs/05 で試してから別に決める。
