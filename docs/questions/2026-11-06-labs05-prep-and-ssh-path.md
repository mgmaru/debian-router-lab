# labs/05 の準備は何のためか、WAN → Client VM の接続は止めるのか、forward を止めると管理用の SSH は切れるのか

調査日：2026-11-06

関連：

- [labs/05-firewall](../../labs/05-firewall/README.md)（実験 0・1 の手順、Notes の「実験 0・1 の目的」「管理用の SSH が通る関所」）
- [関所（Netfilter のフック）](../concepts/kernel/netfilter-hooks.md)（パケットが通る3つの道）
- [ルーターは「返事」と「外からの新しい接続」をどう区別するのか](2026-11-06-reply-vs-new-connection.md)（質問72）
- [ファイアウォール](../concepts/firewall.md)
- [environment/remote-access](../../environment/remote-access/README.md)、[ssh_config.example](../../environment/remote-access/ssh_config.example)

## 疑問

> **質問78**：labs/05 の「0. 準備：戻り道（VMware Console）と、今の状態を確かめる」は何のために行うのか？

> **質問79**：「rule を入れる前に、WAN 側から Client VM へ新しい接続が届いてしまうことを確かめる」とあるが、これをファイアウォールで制限するという認識で良いか？

> **質問80（追加質問）**：Windows から Client VM への接続を通さなくなると、Mac → Windows → Router VM → Client VM の SSH が切れる気がする。そこで、Router VM の input と forward は SSH のみ許可する、という認識で良いか？

## 結論

| 質問 | 答え |
|---|---|
| 実験 0 は何のため？ | **後の実験で困らないようにするためと、比べるための出発点を記録するため。** SSH が切れたときの戻り道（Console）、許可すべき SSH の送信元、今の状態、戻す先（バックアップ）を先に用意する |
| WAN → Client VM の接続をファイアウォールで制限する？ | **はい。** 実験 2 で ③ forward をデフォルト拒否にして止める。ただし止めるのは WAN → LAN の**新しい接続（`new`）だけ**。返事（`established`）は通す |
| forward を止めると、Mac → Client VM の SSH は切れる？ | **切れない。** ProxyJump では、Client VM への接続を始めるのは **Router VM 自身**なので、③ forward を通らない |
| input と forward は SSH のみ許可？ | **いいえ。** input には SSH に加えて返事（`established`）などが要る。forward には SSH の許可は要らず、返事と LAN → WAN の許可が要る |

## 質問78：実験 0 は何のためか

ファイアウォールは、設定を間違えると**自分の接続を切ってしまう**、数少ない設定である。そのため、実験の前に「戻り道」と「戻す先」を用意し、「出発点」を記録しておく。

```text
 実験 0（準備）                         後の実験
 ┌───────────────────────────┐
 │ 0-1 Console にログイン       │──→ 実験 3・4 で SSH が切れたときの戻り道
 │ 0-2 SSH の送信元 IP          │──→ 実験 3 で input に許可する相手
 │ 0-2b Client VM への SSH の送信元│──→ 実験 2 で SSH が切れないことの確認
 │ 0-3 今の ruleset・ip_forward │──→ 実験 1・2 で比べる出発点
 │ 0-4 バックアップ             │──→ 実験 2 以降で、labs/04 の状態に戻す先
 └───────────────────────────┘
```

| 手順 | 何のためか | 用意しないと |
|---|---|---|
| 0-1 Console にログイン | SSH が切れたときの**戻り道**を、切れる前に確かめておく | 締め出されたときに、パスワードを忘れていたなどで戻れない |
| 0-2 `$SSH_CONNECTION`（Router VM） | 管理用の SSH が**どの IP から来ているか**を知る | input の rule で許可する相手を間違え、締め出される |
| 0-2b `$SSH_CONNECTION`（Client VM） | Client VM への SSH が forward を通らないことを確かめる（質問80） | 実験 2 で SSH が切れるかどうかを、予想でしか言えない |
| 0-3 今の状態 | **出発点を記録する**（forward に rule が無い、`ip_forward` が 1） | 実験 1 で「届いた」理由がはっきりしない。実験 2 の後と比べられない |
| 0-4 バックアップ | うまくいかなくなったときに、labs/04 の状態に**一発で戻す** | どこまで戻せばよいかわからなくなる |

## 質問79：WAN → Client VM の接続をファイアウォールで制限するのか

はい。実験 2 で ③ forward をデフォルト拒否にして止める。

ただし、WAN → LAN の向きを**すべて**止めるわけではない。返事も WAN → LAN の向きで届くので、すべて止めると Client VM がインターネットを使えなくなる。conntrack の判定（`ct state`）で、「返事は通す、外から始まった新しい接続は止める」に分ける（質問72）。

```text
                         Router VM の ③ forward
 WAN 側                                                         LAN 側
 Windows ──── ping・SSH（new）──────────→ ✕ 止める
 8.8.8.8 ──── 返事（established）───────→ ○ 通す ─────────────→ Client VM
 8.8.8.8 ←─────────────────────────────── ○ 通す ←── 行き（new）── Client VM
```

| 通信 | 実験 1（rule なし） | 実験 2（forward をデフォルト拒否） |
|---|---|---|
| Windows → Client VM の ping・SSH（WAN → LAN の `new`） | 届く | **止まる** |
| Client VM → 8.8.8.8（LAN → WAN の `new`） | 通る | 通る（rule で許可する） |
| 8.8.8.8 → Client VM の返事（WAN → LAN の `established`） | 通る | 通る（rule で許可する） |

実験 1 は、この**制限をかける前の状態**を見ておくための実験である。同じ確かめ方で「rule を入れる前は届いた → 入れた後は止まった」を比べることで、止まった理由が rule だとはっきり言える。

## 質問80：forward を止めると、管理用の SSH は切れるのか

### ProxyJump では、区間ごとに別々の接続になる

Mac から Client VM への SSH（`ssh debian-client`）は、`ProxyJump windows-home,debian-router` で入っている。この設定では、SSH の接続が**区間ごとに別々の TCP 接続**になる。

```text
 Mac ──(1)──→ Windows ──(2)──→ Router VM ──(3)──→ Client VM
            Tailscale       192.168.22.1 →        10.0.0.1 →
                            192.168.22.129:22     10.0.0.10:22
```

| 区間 | 接続を始めるもの | 送信元 → 宛先 | Router VM の中で通る関所 |
|---|---|---|---|
| (1) Mac → Windows | Mac | Tailscale | Router VM は通らない |
| (2) Windows → Router VM | Windows | 192.168.22.1 → 192.168.22.129:22 | **② input**（Router VM 宛て） |
| (3) Router VM → Client VM | **Router VM の sshd** | 10.0.0.1 → 10.0.0.10:22 | **④ output**（行き）、**② input**（Client VM からの返事） |

(3) の接続を始めるのは Windows ではなく、**Router VM 自身**である。Router VM の sshd が、Mac の代わりに Client VM へつなぎに行く。そのため、このパケットは「通り抜ける」パケット（道1）ではなく、「Router VM から出る」パケット（道3）になる。

### Router VM の中で通る関所

```text
 (2) Windows からの SSH
 ens33 ─→ ① prerouting ─→ ルーティング判断（自分宛て）─→ ② input ─→ sshd
                                                                    │
                                                                    │ Mac の代わりに Client VM へつなぐ
 (3) Client VM への SSH                                               ▼
 sshd ─→ ルーティング判断 ─→ ④ output ─→ ⑤ postrouting ─→ ens37 ─→ Client VM

 (3) の返事
 ens37 ─→ ① prerouting ─→ ルーティング判断（自分宛て）─→ ② input ─→ sshd

                ③ forward はどこにも出てこない
```

### 直接つなぐ SSH との違い

実験 1 の Windows からの `ssh kajih@10.0.0.10` は、Windows が Client VM へ**直接**つなぎに行く通信である。こちらは Router VM を通り抜けるので、③ forward を通る。

```text
 ProxyJump（ssh debian-client）
   Windows ═══ TCP ═══→ Router VM の sshd ═══ 別の TCP ═══→ Client VM
                        （② input）           （④ output）

 直接（Windows から ssh kajih@10.0.0.10）
   Windows ═══════════ 1本の TCP ═══════════════════════→ Client VM
                        Router VM は通り抜けるだけ（③ forward）
```

| 通信 | Client VM から見た接続相手 | 通る関所 | 実験 2 の後 |
|---|---|---|---|
| `ssh debian-client`（ProxyJump） | **10.0.0.1**（Router VM） | ② input と ④ output | **切れない** |
| Windows から `ssh kajih@10.0.0.10`（直接） | **192.168.22.1**（Windows） | ③ forward | **止まる** |

実験 2 で止めたいのは、直接つなぐほうだけである。

### input と forward は「SSH のみ許可」ではない

| chain | 許可が要るもの | 「SSH のみ許可」にすると |
|---|---|---|
| ② input | ・(2) の SSH（192.168.22.1 からの TCP/22）<br>・`ct state established,related`（(3) の Client VM からの返事を含む）<br>・loopback・ICMP など（実験 3 で試す） | (3) の返事が止まり、**Client VM への SSH が切れる**。Router VM の `apt update` も返事が来なくなる |
| ③ forward | ・`ct state established,related`<br>・LAN → WAN の新しい接続（`iifname "ens37" oifname "ens33"`） | **Client VM がインターネットを使えなくなる**。forward に SSH の許可は要らない |

input の `established` は見落としやすい。(3) で Router VM が Client VM に送ったパケットの返事は、**Router VM 宛て**に届くので ② input を通る。input をデフォルト拒否にするとき、SSH（TCP/22 宛て）の許可だけでは、この返事（Client VM の 22 番**から**来るパケット）は通らない。

### 確かめ方

labs/05 の実験 0 に「0-2b」として追加した。`ssh debian-client` で Client VM に入り、`echo $SSH_CONNECTION` を打つ。送信元が `192.168.22.1` ではなく **`10.0.0.1`** と出れば、(3) の接続を Router VM が始めている証拠になる。
