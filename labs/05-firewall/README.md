# 05 - Firewall

## Goal

通信の許可・拒否を制御する。

例：

```text
LAN → Internet : Allow
Internet → LAN : Deny
SSH → Router   : 条件付きAllow
```

利用：

```text
nftables
```

## Notes

Router VMへの管理用SSHはWAN側NIC（VMnet8）を通っている。

WAN側のTCP/22を誤って遮断するとSSHできなくなるため、VMware Consoleを復旧手段として残しておく（[environment/remote-access](../../environment/remote-access/README.md) を参照）。

ファイアウォールの rule は ② input（Router VM を守る）と ③ forward（LAN 側の機器を守る）に置く（[ADR 008](../../docs/decisions/008-nftables-hook-placement.md)）。

rule の評価のしかたとデフォルト拒否の考え方は [docs/concepts/firewall.md](../../docs/concepts/firewall.md) に、関所は [docs/concepts/kernel/netfilter-hooks.md](../../docs/concepts/kernel/netfilter-hooks.md) に、返事と新しい接続の区別（`ct state`）は [docs/concepts/kernel/conntrack.md](../../docs/concepts/kernel/conntrack.md) と [質問72](../../docs/questions/2026-11-06-reply-vs-new-connection.md) にまとめた。

### 実験の計画

| 実験 | 内容 | 締め出される危険 |
|---|---|---|
| 0 | 準備：戻り道（VMware Console）と、今の状態を確かめる | なし |
| 1 | rule を入れる前に、WAN 側から Client VM へ新しい接続が届いてしまうことを確かめる | なし |
| 2 | forward をデフォルト拒否にする | なし（forward は Router VM 宛ての SSH を通らない） |
| 3 | input をデフォルト拒否にする | **あり**（順番を間違えると SSH が切れる） |
| 4 | `/etc/nftables.conf` に書いて永続化し、再起動後も効くか確かめる | あり |

実験 2 以降の手順は、実験 0・1 の結果を見てから書き足す。実験 3 の締め出し対策（一定時間後に rule を自動で消す仕掛けなど）も、そのときに決める。

## Topology

```text
                        WAN側（VMnet8：192.168.22.0/24）                     LAN側（VMnet2：10.0.0.0/24）

 Mac ─(Tailscale)─ Windows ──────────────── ens33（192.168.22.129）
                   192.168.22.1（予想）          Router VM
                                              ens37（10.0.0.1） ──────────────── Client VM（10.0.0.10）
                   VMware NAT ───── インターネット
                   192.168.22.2
```

- Mac から Router VM・Client VM への SSH は、Windows を踏み台にしている。そのため Router VM から見ると、管理用の SSH は **Windows の VMnet8 側の IP（192.168.22.1 の予想）から WAN 側に来る**通信である
- この実験では、Windows を「WAN 側にいる、外の機器」として使う

## Experiment

### 0. 準備：戻り道と、今の状態を確かめる

#### 0-1. VMware Console から Router VM にログインできることを確かめる（Windows）

1. Windows の VMware Workstation で Router VM のタブを開く
2. Console 画面で、ユーザー名とパスワードでログインする
3. `hostname` を打ち、`debian-router` と出ることを確かめる
4. `exit` でログアウトする

予想：SSH を使わずに Router VM のシェルに入れる。実験 3 以降で SSH が切れたときは、ここから rule を消して戻す。

#### 0-2. 管理用の SSH が、どこから来ているかを確かめる（Router VM）

Mac から `ssh debian-router` で入った状態で、次を打つ。

```bash
echo $SSH_CONNECTION
```

`$SSH_CONNECTION` には、「送信元 IP、送信元ポート、宛先 IP、宛先ポート」の4つが入っている。

予想：

```text
192.168.22.1 <ポート番号> 192.168.22.129 22
```

| 欄 | 予想 | 意味 |
|---|---|---|
| 送信元 IP | 192.168.22.1 | Windows の VMnet8 側の IP。input で許可する必要がある相手 |
| 宛先 IP | 192.168.22.129 | Router VM の ens33（WAN 側） |
| 宛先ポート | 22 | SSH |

送信元 IP が予想と違ったら、その値を記録する。実験 3 の rule で使う。

#### 0-3. 今の nftables・転送の状態を確かめる（Router VM）

```bash
sudo nft list ruleset
systemctl is-enabled nftables
sysctl net.ipv4.ip_forward
```

予想：

| コマンド | 予想 |
|---|---|
| `sudo nft list ruleset` | labs/04 で入れた状態。`table inet filter` の input・forward・output は空（rule なし、policy なし）、`table ip nat` に `oifname "ens33" masquerade` |
| `systemctl is-enabled nftables` | `enabled` |
| `sysctl net.ipv4.ip_forward` | `net.ipv4.ip_forward = 1` |

#### 0-4. 今の設定ファイルをバックアップする（Router VM）

```bash
sudo cp /etc/nftables.conf /etc/nftables.conf.labs04
ls -l /etc/nftables.conf*
```

予想：`/etc/nftables.conf` と `/etc/nftables.conf.labs04` の2つが並ぶ。うまくいかなくなったら、`sudo nft -f /etc/nftables.conf.labs04` で labs/04 の状態に戻せる。

---

### 1. rule を入れる前に、WAN 側から Client VM へ届くか確かめる

[質問72](../../docs/questions/2026-11-06-reply-vs-new-connection.md) で「今の Router VM は forward に rule を何も置いていないので、WAN 側から Client VM へ新しい接続が届くはず」と書いた。これを確かめる。

#### 1-1. Windows の VMnet8 側の IP を確かめる（Windows）

コマンドプロンプトか PowerShell で次を打つ。

```powershell
ipconfig
```

予想：`イーサネット アダプター VMware Network Adapter VMnet8` の `IPv4 アドレス` が `192.168.22.1`。

#### 1-2. ルートを足す前に、Windows から Client VM へ ping を送る（Windows）

```powershell
ping 10.0.0.10
```

予想：`要求がタイムアウトしました。` になる。Windows は 10.0.0.0/24 が Router VM の向こうにあることを知らないので、自分のデフォルトゲートウェイ（家のルーター）に送ってしまい、届かない。

もし家のネットワーク自体が `10.0.0.0/24` を使っている場合は、家の機器と区別できなくなるので、実験の前に相談する（1-1 の `ipconfig` で、Wi-Fi や有線のアダプターの IP を確かめる）。

#### 1-3. Windows に「10.0.0.0/24 は Router VM 経由」というルートを足す（Windows・管理者）

**管理者として**開いたコマンドプロンプトか PowerShell で次を打つ。

```powershell
route add 10.0.0.0 mask 255.255.255.0 192.168.22.129
route print 10.0.0.0
```

予想：`route print` の「アクティブ ルート」に、`10.0.0.0  255.255.255.0  192.168.22.129` の行が出る。

- `-p` を付けていないので、このルートは一時的なもの。Windows を再起動すると消える
- 実験 2 でも同じ確かめ方をするので、labs/05 が終わるまで消さない

#### 1-4. Router VM で tcpdump を動かす（Router VM、ターミナル2つ）

ターミナル1（WAN 側）：

```bash
sudo tcpdump -n -i ens33 host 10.0.0.10
```

ターミナル2（LAN 側）：

```bash
sudo tcpdump -n -i ens37 host 192.168.22.1
```

#### 1-5. Windows から Client VM へ ping と SSH を送る（Windows）

```powershell
ping 10.0.0.10
ssh kajih@10.0.0.10
```

SSH はパスワードを聞かれたところ（またはログインできたところ）で止めてよい。ログインできた場合は、1-7 を Client VM 側で確かめてから `exit` する。

予想：

| 通信 | 予想 | 理由 |
|---|---|---|
| `ping 10.0.0.10` | **返事が来る** | forward に rule が無いので、WAN → LAN の新しい接続も素通りする |
| `ssh kajih@10.0.0.10` | **Client VM のパスワードを聞かれる** | 同上。外から Client VM の SSH に直接つながる |

tcpdump の予想：

| NIC | 見えるパケット |
|---|---|
| ens33（WAN 側） | `192.168.22.1 > 10.0.0.10`（行き）と `10.0.0.10 > 192.168.22.1`（返事） |
| ens37（LAN 側） | 同じ2種類。**送信元・宛先は書き換わっていない** |

返事の送信元が `10.0.0.10` のまま ens33 から出ていくのは、マスカレードの rule（`oifname "ens33"`）が、**通信の最初のパケット**の出口で決まるからである。最初のパケット（Windows → Client VM）の出口は ens37 なので、マスカレードは使われない（[netfilter-hooks.md の「NAT の chain を通るのは最初のパケットだけ」](../../docs/concepts/kernel/netfilter-hooks.md#1つだけ大きな違いnat-の-chain-を通るのは最初のパケットだけ)）。

#### 1-6. conntrack の記録を見る（Router VM）

ping を打っている間に、次を打つ。

```bash
sudo conntrack -L | grep 10.0.0.10
```

予想（ICMP の行）：

```text
icmp     1 29 src=192.168.22.1 dst=10.0.0.10 type=8 code=0 id=... src=10.0.0.10 dst=192.168.22.1 type=0 code=0 id=... mark=0 use=1
```

| 部分 | 予想 | 意味 |
|---|---|---|
| 前半（行きの向き） | `src=192.168.22.1 dst=10.0.0.10` | **外（WAN 側）から始まった**通信として記録されている |
| 後半（返事の向き） | `src=10.0.0.10 dst=192.168.22.1` | 前半をそのまま逆にした形。NAT による書き換えが無い |

labs/04 で見た行（Client VM が始めた通信）は、前半の送信元が `10.0.0.10` で、後半の宛先が `192.168.22.129` に書き換わっていた。向きと書き換えの有無を比べる。

#### 1-7. Client VM から見た接続相手を確かめる（Client VM）

1-5 の SSH でログインできた場合、その SSH の中で（または別の SSH で Client VM に入って）次を打つ。

```bash
ss -tn state established '( sport = :22 )'
```

予想：`Peer Address:Port` の欄に `192.168.22.1:<ポート番号>` が出る。Client VM は、**WAN 側の機器と直接つながっている**ことになる。

#### 実験 1 で確かめたいこと

| 確かめたいこと | 予想 |
|---|---|
| NAT（マスカレード）だけで、外からの新しい接続は止まるか | **止まらない**。WAN 側の機器が 10.0.0.0/24 へのルートを持っていれば、Client VM まで届く |
| 外から始まった通信は、conntrack にどう記録されるか | 前半の送信元が WAN 側の IP。NAT の書き換えは無い |
| 実験 2 で止めたいパケット | 1-5 の `ping 10.0.0.10` と `ssh kajih@10.0.0.10`（WAN → LAN の `new`） |

## Result

（実験を実行したら、結果をここに記録する）
