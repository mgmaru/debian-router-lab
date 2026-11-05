# 04 - NAT

## Goal

プライベートネットワークから外部ネットワークへ通信するためにNATを設定する。

```text
10.0.0.10
    ↓
Debian Router
    ↓
Source NAT
    ↓
WAN IP
    ↓
Internet
```

利用：

```text
nftables
```

## Notes

クライアント VM の `10.0.0.10` は、外（VMware の NAT 側）からは見えないアドレスである。そのまま外に出すと、返事が戻ってこない。

ルーター VM が送信元を自分の外側の IP（`192.168.22.129`）に書き換えて送り出し、返事を受け取ったらクライアント VM に戻す、という処理（**NAT / マスカレード**）が必要になる。

nftables で設定し、`/etc/nftables.conf` ＋ `systemctl enable nftables` で永続化する。

NAT の仕組みと、NAT が無いときに返事が戻らない理由は [docs/concepts/nat.md](../../docs/concepts/nat.md) に、nftables の table・chain・rule と関所（フック）は [docs/concepts/nftables.md](../../docs/concepts/nftables.md) に、`nft` コマンドの形式は [docs/tools/nft.md](../../docs/tools/nft.md) にまとめた。

## Topology

```text
Client VM                       Router VM                                 VMware NAT
10.0.0.10  ──→  ens37（10.0.0.1）        ens33（192.168.22.129）  ──→  192.168.22.2  ──→  インターネット（8.8.8.8）
                  LAN側                     WAN側
```

## Experiment

### 1. NAT が無い状態で、WAN へ ping を送る（Router VM / Client VM）

Client VM から `ping 8.8.8.8` を打ち、Router VM の ens37（LAN側）と ens33（WAN側）で `tcpdump` を取った（2026-11-05）。

Router VM ens37（LAN側）：

```text
kajih@debian-router:~$ sudo tcpdump -n -i ens37 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens37, link-type EN10MB (Ethernet), snapshot length 262144 bytes
00:20:23.334156 IP 10.0.0.10 > 8.8.8.8: ICMP echo request, id 10, seq 1, length 64
00:20:24.360163 IP 10.0.0.10 > 8.8.8.8: ICMP echo request, id 10, seq 2, length 64
00:20:25.383640 IP 10.0.0.10 > 8.8.8.8: ICMP echo request, id 10, seq 3, length 64
（seq 20 まで、echo request だけが続いた）
```

Router VM ens33（WAN側）：

```text
kajih@debian-router:~$ sudo tcpdump -n -i ens33 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens33, link-type EN10MB (Ethernet), snapshot length 262144 bytes
00:23:19.461296 IP 10.0.0.10 > 8.8.8.8: ICMP echo request, id 12, seq 1, length 64
00:23:20.486323 IP 10.0.0.10 > 8.8.8.8: ICMP echo request, id 12, seq 2, length 64
00:23:21.510702 IP 10.0.0.10 > 8.8.8.8: ICMP echo request, id 12, seq 3, length 64
```

### 2. Router VM のデフォルトゲートウェイを確認する（Router VM）

```text
kajih@debian-router:~$ ip route show default
default via 192.168.22.2 dev ens33 proto dhcp src 192.168.22.129 metric 1002
```

### 3. nftables の状態を確認する（Router VM）

```text
kajih@debian-router:~$ nft --version
-bash: nft: command not found
kajih@debian-router:~$ sudo nft --version
nftables v1.1.3 (Commodore Bullmoose #4)
kajih@debian-router:~$ sudo nft list ruleset
kajih@debian-router:~$ systemctl status nftables
○ nftables.service - nftables
     Loaded: loaded (/usr/lib/systemd/system/nftables.service; disabled; pres>
     Active: inactive (dead)
       Docs: man:nft(8)
             http://wiki.nftables.org
```

`sudo nft list ruleset` は何も表示しなかった。

### 4. ほかのツールの rule と、関所の状態を確認する（Router VM）

```text
kajih@debian-router:~$ sudo iptables --version
sudo: iptables: command not found
kajih@debian-router:~$ sudo iptables -L -n
sudo: iptables: command not found
kajih@debian-router:~$ sudo nft list hooks
family ip {
	hook postrouting {
		-0000000225 apparmor_ip_postroute
	}
}
family ip6 {
	hook postrouting {
		-0000000225 apparmor_ip_postroute
	}
}
```

### 5. table・chain・rule を1つずつ作る（Router VM）

1つ作るたびに `sudo nft list ruleset` で中身を確かめた（2026-11-05）。

table を作る：

```text
kajih@debian-router:~$ sudo nft add table ip nat
kajih@debian-router:~$ sudo nft list ruleset
table ip nat {
}
```

chain を作り、postrouting の関所に置く：

```text
kajih@debian-router:~$ sudo nft add chain ip nat postrouting '{ type nat hook postrouting priority srcnat; policy accept; }'
kajih@debian-router:~$ sudo nft list hooks
family ip {
	hook prerouting {
		-0000000100 nf_nat_ipv4_pre_routing [nf_nat]
	}
	hook input {
		+0000000100 nf_nat_ipv4_local_in [nf_nat]
	}
	hook output {
		-0000000100 nf_nat_ipv4_local_fn [nf_nat]
	}
	hook postrouting {
		-0000000225 apparmor_ip_postroute
		+0000000100 nf_nat_ipv4_out [nf_nat]
	}
}
family ip6 {
	hook postrouting {
		-0000000225 apparmor_ip_postroute
	}
}
kajih@debian-router:~$ sudo nft list ruleset
table ip nat {
	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
	}
}
```

最初は閉じの `'` を付け忘れて、シェルが続きの入力を待つ `>` が出た。`Ctrl + C` で取り消して打ち直した。

rule を足す：

```text
kajih@debian-router:~$ sudo nft add rule ip nat postrouting oifname "ens33" masquerade
kajih@debian-router:~$ sudo nft list ruleset
table ip nat {
	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		oifname "ens33" masquerade
	}
}
kajih@debian-router:~$ sudo nft list hooks
family ip {
	hook prerouting {
		-0000000400 ipv4_conntrack_defrag [nf_defrag_ipv4]
		-0000000200 ipv4_conntrack_in [nf_conntrack]
		-0000000100 nf_nat_ipv4_pre_routing [nf_nat]
	}
	hook input {
		+0000000100 nf_nat_ipv4_local_in [nf_nat]
		+2147483647 nf_confirm [nf_conntrack]
	}
	hook output {
		-0000000400 ipv4_conntrack_defrag [nf_defrag_ipv4]
		-0000000200 ipv4_conntrack_local [nf_conntrack]
		-0000000100 nf_nat_ipv4_local_fn [nf_nat]
	}
	hook postrouting {
		-0000000225 apparmor_ip_postroute
		+0000000100 nf_nat_ipv4_out [nf_nat]
		+2147483647 nf_confirm [nf_conntrack]
	}
}
family ip6 {
	hook postrouting {
		-0000000225 apparmor_ip_postroute
	}
}
```

### 6. マスカレードを入れた状態で、WAN へ ping を送る（Router VM / Client VM）

Router VM で ens37 と ens33 の `tcpdump` を別のターミナルで動かしたまま、Client VM から `ping -c 3 8.8.8.8` を打った（2026-11-05）。

Client VM：

```text
kajih@debian-client:~$ ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=127 time=14.8 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=127 time=13.6 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=127 time=12.8 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 12.772/13.721/14.817/0.841 ms
```

Router VM ens37（LAN側）：

```text
kajih@debian-router:~$ sudo tcpdump -n -e -v -i ens37 icmp
tcpdump: listening on ens37, link-type EN10MB (Ethernet), snapshot length 262144 bytes
13:55:30.544482 00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 11806, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.10 > 8.8.8.8: ICMP echo request, id 13, seq 1, length 64
13:55:30.558949 00:0c:29:73:2d:6b > 00:0c:29:20:6c:ff, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 127, id 35149, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 10.0.0.10: ICMP echo reply, id 13, seq 1, length 64
13:55:31.546472 00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 11855, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.10 > 8.8.8.8: ICMP echo request, id 13, seq 2, length 64
13:55:31.559014 00:0c:29:73:2d:6b > 00:0c:29:20:6c:ff, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 127, id 16890, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 10.0.0.10: ICMP echo reply, id 13, seq 2, length 64
13:55:32.548064 00:0c:29:20:6c:ff > 00:0c:29:73:2d:6b, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 12027, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.10 > 8.8.8.8: ICMP echo request, id 13, seq 3, length 64
13:55:32.559976 00:0c:29:73:2d:6b > 00:0c:29:20:6c:ff, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 127, id 50337, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 10.0.0.10: ICMP echo reply, id 13, seq 3, length 64
```

Router VM ens33（WAN側）：

```text
kajih@debian-router:~$ sudo tcpdump -n -e -v -i ens33 icmp
tcpdump: listening on ens33, link-type EN10MB (Ethernet), snapshot length 262144 bytes
13:55:30.544608 00:0c:29:73:2d:61 > 00:50:56:f9:12:3d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 63, id 11806, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.22.129 > 8.8.8.8: ICMP echo request, id 13, seq 1, length 64
13:55:30.558932 00:50:56:f9:12:3d > 00:0c:29:73:2d:61, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 128, id 35149, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.22.129: ICMP echo reply, id 13, seq 1, length 64
13:55:31.546513 00:0c:29:73:2d:61 > 00:50:56:f9:12:3d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 63, id 11855, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.22.129 > 8.8.8.8: ICMP echo request, id 13, seq 2, length 64
13:55:31.558935 00:50:56:f9:12:3d > 00:0c:29:73:2d:61, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 128, id 16890, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.22.129: ICMP echo reply, id 13, seq 2, length 64
13:55:32.548108 00:0c:29:73:2d:61 > 00:50:56:f9:12:3d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 63, id 12027, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.22.129 > 8.8.8.8: ICMP echo request, id 13, seq 3, length 64
13:55:32.559925 00:50:56:f9:12:3d > 00:0c:29:73:2d:61, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 128, id 50337, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.22.129: ICMP echo reply, id 13, seq 3, length 64
```

## Result

### 1. NAT が無い状態で、WAN へ ping を送る（Router VM / Client VM）

| NIC | echo request | echo reply | 送信元 IP |
|---|---|---|---|
| ens37（LAN側） | 出た | **出なかった** | `10.0.0.10` |
| ens33（WAN側） | 出た | **出なかった** | `10.0.0.10`（**書き換わっていない**） |

- ens33 に echo request が出ているので、IP フォワードは効いている（labs/02 で有効にした設定）。Router VM は、ルーティングテーブルの default の行に従って `192.168.22.2` へ渡している（Experiment 2）
- 送信元は `10.0.0.10` のまま外へ出ている。labs/03 で確かめた「転送するパケットは送信元が書き換わらない」を、WAN 側でも確かめられた
- echo reply は ens33 にも出ていない。`tcpdump` は、NIC にパケットが届いた時点で表示するので、**返事は Router VM まで戻ってきていない**。Router VM が返事を捨てたのではない
- 外側（VMware NAT から先）は `10.0.0.0/24` を知らない。また `10.0.0.10` はプライベートアドレスなので、インターネットでは返事を運べない。そのため、送信元を `10.0.0.10` のまま出すと返事は戻らない。これが NAT が必要な理由（[docs/concepts/nat.md](../../docs/concepts/nat.md)）

#### 最初に LAN 側で見てしまったこと

最初は WAN 側のつもりで ens37 を指定していたが、ens37 は LAN 側だった。送信元が外へ出るときにどうなっているかは、**WAN 側（ens33）で見ないとわからない**。

### 2. Router VM のデフォルトゲートウェイを確認する（Router VM）

| 部分 | 意味 |
|---|---|
| `via 192.168.22.2` | デフォルトゲートウェイ（VMware NAT） |
| `dev ens33` | 送り出す NIC（WAN側） |
| `proto dhcp` | VMware の DHCP からもらった行 |
| `src 192.168.22.129` | Router VM **自身が**送るときに使う送信元 IP。転送する Client VM のパケットには使われない |
| `metric 1002` | 同じ宛先の行が複数あるときの優先度。小さいほど優先される |

Client VM のパケットにも `192.168.22.129` を使わせるのが、これから設定するマスカレードである。

### 3. nftables の状態を確認する（Router VM）

| 確認したこと | 結果 | 意味 |
|---|---|---|
| `nft --version` | `command not found` | `nft` は `/usr/sbin/nft` にあり、`/usr/sbin` は一般ユーザーの PATH に無い。**`nft` は `sudo` を付けて実行する** |
| `sudo nft --version` | `nftables v1.1.3` | nftables は入っていた（Debian 13 のバージョン） |
| `sudo nft list ruleset` | 何も出ない | ルール（table）が1つも無い。どの関所も素通りしている |
| `systemctl status nftables` | `disabled`、`inactive (dead)` | 起動時に `/etc/nftables.conf` を読み込む設定になっておらず、今も読み込まれていない |

今の Router VM は、何も書き換えず、何も止めずに転送している。

### 4. ほかのツールの rule と、関所の状態を確認する（Router VM）

- `iptables` は `sudo` を付けても `command not found`。**iptables 自体が入っていない**ので、iptables の rule がぶつかる心配はない
- `nft list hooks` には、IPv4・IPv6 とも postrouting の関所に `apparmor_ip_postroute`（priority `-225`）だけが出た
  - AppArmor は Debian に最初から入っているセキュリティの仕組みで、その確認のためにカーネルが登録している。パケットを書き換えたり捨てたりはしない
  - 自分の chain はまだ無いので、ほかの関所には何も出ていない。「何も登録されていなければ素通り」の状態
- **予想と違った**：conntrack が出ていない。今の Linux では、conntrack は必要になるまで関所に登録されないためと考えられる。NAT の chain を作った後に、conntrack や NAT の処理が現れるかを確かめる

### 5. table・chain・rule を1つずつ作る（Router VM）

- `sudo nft add table ip nat` で、IPv4 用の table「nat」ができた。中身（chain）はまだ空
- table は関所に置くものではないので、この時点ではまだどの関所にも何も登録されていない
- chain を作ると、`list ruleset` の table の中に chain が増えた。rule はまだ無い
- `list hooks` では、**予想と違って**、自分の chain の名前ではなく、カーネルの NAT の部品（`[nf_nat]`）が出てきた。`type nat` の chain は関所に直接登録されず、`nf_nat` が関所に登録され、そこから自分の chain が呼ばれる
- postrouting だけに chain を作ったのに、NAT の部品は **4つの関所**に登録された

| 関所 | 登録された処理 | priority | 役割 |
|---|---|---|---|
| prerouting | `nf_nat_ipv4_pre_routing` | -100（dstnat） | 届いたパケットの宛先を書き換える。**帰りの書き戻しはここ** |
| input | `nf_nat_ipv4_local_in` | +100 | Router VM 自身宛てのパケットの NAT |
| output | `nf_nat_ipv4_local_fn` | -100 | Router VM 自身が送るパケットの NAT |
| postrouting | `nf_nat_ipv4_out` | +100（srcnat） | 出ていくパケットの送信元を書き換える。**マスカレードはここ** |

- postrouting の `+100` は、chain を作るときに指定した `priority srcnat`（= 100）と同じ
- 帰りの返事の宛先を書き戻す処理が、prerouting に登録されているのが見えた。帰り用の rule を自分で書かなくてよいのは、このため
- conntrack の処理は、まだ出ていない。rule を入れた後に確かめる
- `family ip6` は変わらなかった。作ったのは `table ip`（IPv4 用）だから
- rule を足すと、`list ruleset` の chain の中に `oifname "ens33" masquerade` が増えた
- `list hooks` では、chain だけのときには無かった **conntrack の処理が現れた**。`masquerade` は帰りの書き戻しに conntrack の記録が必要なので、rule を入れた時点でカーネルが conntrack を関所に登録した

| 処理 | 関所 | priority | 役割 |
|---|---|---|---|
| `ipv4_conntrack_defrag` | prerouting、output | -400 | 分割されて届いたパケットを組み立て直す。conntrack がポート番号などを読めるようにするため |
| `ipv4_conntrack_in` | prerouting | -200 | 届いたパケットが記録済みの通信のものかを調べる。新しい通信なら記録を作り始める |
| `ipv4_conntrack_local` | output | -200 | 同じことを、Router VM 自身が送るパケットに対して行う |
| `nf_confirm` | input、postrouting | +2147483647（最大） | いちばん最後に、新しい通信の記録を確定する。途中で捨てられたパケットの記録は残さない |

- prerouting の中は「組み立て直す（-400）→ conntrack で記録を調べる（-200）→ NAT で宛先を書き戻す（-100）→ ルーティング判断」の順。conntrack が NAT より先なので、NAT は記録を使って書き戻せる。全部がルーティング判断より前なので、書き戻した宛先 `10.0.0.10` で ens37 への経路が選ばれる
- forward には何も登録されていない。転送するパケットも prerouting と postrouting を通るので、そこで conntrack と NAT が済む

### 6. マスカレードを入れた状態で、WAN へ ping を送る（Router VM / Client VM）

**Client VM から 8.8.8.8 へ ping が通った**（3 packets transmitted, 3 received）。Experiment 1 では返事が1つも来なかった。

行き（echo request、seq 1）：

| | ens37（LAN 側に届いた） | ens33（WAN 側から出た） | 変わったか |
|---|---|---|---|
| 送信元 MAC → 宛先 MAC | `00:0c:29:20:6c:ff` → `00:0c:29:73:2d:6b` | `00:0c:29:73:2d:61` → `00:50:56:f9:12:3d` | **付け替え** |
| 送信元 IP | `10.0.0.10` | **`192.168.22.129`** | **マスカレードで書き換え** |
| 宛先 IP | `8.8.8.8` | `8.8.8.8` | そのまま |
| TTL | 64 | 63 | 転送で1減った |
| IP ヘッダーの id | 11806 | 11806 | **そのまま** |
| ICMP の id・seq | 13・1 | 13・1 | そのまま |

帰り（echo reply、seq 1）：

| | ens33（WAN 側に届いた） | ens37（LAN 側から出た） | 変わったか |
|---|---|---|---|
| 送信元 IP | `8.8.8.8` | `8.8.8.8` | そのまま |
| 宛先 IP | `192.168.22.129` | **`10.0.0.10`** | **conntrack の記録で書き戻し** |
| TTL | 128 | 127 | 転送で1減った |
| IP ヘッダーの id | 35149 | 35149 | **そのまま** |
| 時刻 | 13:55:30.558932 | 13:55:30.558949 | 約 0.017 ミリ秒で書き戻して転送した |

- 行きは送信元が `192.168.22.129` に書き換わり、帰りは宛先が `10.0.0.10` に書き戻された。帰り用の rule は書いていないので、Experiment 5 で見た prerouting の conntrack と `nf_nat` が書き戻したことになる
- IP ヘッダーの id が Router VM の前後で同じ。IP ヘッダーは作り直されず、同じヘッダーの欄が書き換えられている（[質問63](../../docs/questions/2026-11-05-nat-rewrites-header-field.md)）。MAC アドレスは付け替えられている
- ICMP の id（13）は書き換わらなかった。Client が1台なので、番号をずらす必要がなかったため

#### 予想していなかったこと：帰りの TTL が 128

ens33 に届いた返事の TTL が 128 だった。8.8.8.8 から何台ものルーターを通ってきたなら、128 より小さくなっているはずである。128 は Windows が最初に付ける値なので、**VMware NAT（Windows 上で動いている）が返事のパケットを作り直して Router VM に渡している**可能性が高い。Router VM の中からは確かめきれないので、「そう考えられる」という段階である。
