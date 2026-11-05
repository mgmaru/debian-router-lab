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

NAT の仕組みと、NAT が無いときに返事が戻らない理由は [docs/concepts/nat.md](../../docs/concepts/nat.md) に、nftables の table・chain・rule と関所（フック）は [docs/concepts/nftables.md](../../docs/concepts/nftables.md) に、`nft` コマンドの形式は [docs/tools/nft.md](../../docs/tools/nft.md) に、conntrack の記録の仕組みは [docs/concepts/conntrack.md](../../docs/concepts/conntrack.md) にまとめた。

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

### 7. conntrack の記録を見る（Router VM / Client VM）

Router VM に `conntrack` コマンドを入れ（`sudo apt install conntrack`）、Client VM から `ping 8.8.8.8` を送り続けている間に、Router VM で記録を表示した（2026-11-05）。

```text
kajih@debian-router:~$ sudo conntrack -L
tcp      6 431999 ESTABLISHED src=192.168.22.1 dst=192.168.22.129 sport=53618 dport=22 src=192.168.22.129 dst=192.168.22.1 sport=22 dport=53618 [ASSURED] mark=0 use=1
tcp      6 431999 ESTABLISHED src=10.0.0.1 dst=10.0.0.10 sport=50372 dport=22 src=10.0.0.10 dst=10.0.0.1 sport=22 dport=50372 [ASSURED] mark=0 use=1
udp      17 6 src=192.168.22.129 dst=192.168.22.254 sport=68 dport=67 src=192.168.22.254 dst=192.168.22.129 sport=67 dport=68 mark=0 use=1
tcp      6 299 ESTABLISHED src=192.168.22.129 dst=192.168.22.1 sport=22 dport=57752 src=192.168.22.1 dst=192.168.22.129 sport=57752 dport=22 [ASSURED] mark=0 use=1
icmp     1 29 src=10.0.0.10 dst=8.8.8.8 type=8 code=0 id=14 src=8.8.8.8 dst=192.168.22.129 type=0 code=0 id=14 mark=0 use=1
tcp      6 431685 ESTABLISHED src=192.168.22.1 dst=192.168.22.129 sport=53624 dport=22 src=192.168.22.129 dst=192.168.22.1 sport=22 dport=53624 [ASSURED] mark=0 use=1
tcp      6 431737 ESTABLISHED src=192.168.22.1 dst=192.168.22.129 sport=53619 dport=22 src=192.168.22.129 dst=192.168.22.1 sport=22 dport=53619 [ASSURED] mark=0 use=1
conntrack v1.4.8 (conntrack-tools): 7 flow entries have been shown.
```

### 8. traceroute で経路を見る（Client VM）

labs/03 でやり残した traceroute を、マスカレードを入れた状態で試した（2026-11-05）。`-I` で、ICMP の echo request を使う。

```text
kajih@debian-client:~$ traceroute -I 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  10.0.0.1 (10.0.0.1)  0.818 ms  0.780 ms  0.763 ms
 2  192.168.22.2 (192.168.22.2)  1.091 ms * *
 3  * 192.168.11.1 (192.168.11.1)  2.406 ms  2.389 ms
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  8.8.8.8 (8.8.8.8)  11.619 ms  16.055 ms  16.036 ms
```

### 9. 永続化する：初期状態の `/etc/nftables.conf` を確認する（Router VM）

```text
kajih@debian-router:/etc$ cat nftables.conf
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
	chain input {
		type filter hook input priority filter;
	}
	chain forward {
		type filter hook forward priority filter;
	}
	chain output {
		type filter hook output priority filter;
	}
}
```

### 10. 永続化する：`/etc/nftables.conf` に NAT を書き、サービスを有効にする（Router VM）

初期状態の `table inet filter` はそのまま残し、いちばん下に NAT の table を足した（2026-11-05）。足した部分は、`sudo nft list ruleset` の出力と同じ形である。元のファイルのコピー（`.orig`）は取らなかった（初期状態の中身は Experiment 9 に記録してある）。

```text
kajih@debian-router:/etc$ cat nftables.conf
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
	chain input {
		type filter hook input priority filter;
	}
	chain forward {
		type filter hook forward priority filter;
	}
	chain output {
		type filter hook output priority filter;
	}
}

table ip nat {
	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		oifname "ens33" masquerade
	}
}
```

書き方だけを確かめてから、サービスを有効にした：

```text
kajih@debian-router:/etc$ sudo nft -c -f /etc/nftables.conf
kajih@debian-router:/etc$ sudo systemctl enable --now nftables
kajih@debian-router:/etc$ systemctl status nftables
● nftables.service - nftables
     Loaded: loaded (/usr/lib/systemd/system/nftables.service; enabled; preset: enabled)
     Active: active (exited) since Thu 2026-11-05 14:45:31 JST; 12s ago
 Invocation: 15b97e67fdef4548b063ec0fa33a5c7f
       Docs: man:nft(8)
             http://wiki.nftables.org
    Process: 27228 ExecStart=/usr/sbin/nft -f /etc/nftables.conf (code=exited, status=0/SUCCESS)
   Main PID: 27228 (code=exited, status=0/SUCCESS)
   Mem peak: 2M
        CPU: 20ms
kajih@debian-router:/etc$ sudo nft list ruleset
table inet filter {
	chain input {
		type filter hook input priority filter; policy accept;
	}

	chain forward {
		type filter hook forward priority filter; policy accept;
	}

	chain output {
		type filter hook output priority filter; policy accept;
	}
}
table ip nat {
	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		oifname "ens33" masquerade
	}
}
```

### 11. 再起動しても NAT が効くか確かめる（Router VM / Client VM）

Router VM を `sudo reboot` で再起動した後、Client VM から `ping -c 3 8.8.8.8` を打った（2026-11-05）。

```text
kajih@debian-client:~$ ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=127 time=11.9 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=127 time=12.4 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=127 time=14.5 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 11.851/12.909/14.466/1.124 ms
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

### 7. conntrack の記録を見る（Router VM / Client VM）

ping の行は、予想どおり NAT されていた。

| 部分 | 値 | 意味 |
|---|---|---|
| 残り秒数 | 29 | ping が続いている間は、届くたびに 30 秒に戻る |
| 前半（行き） | `src=10.0.0.10 dst=8.8.8.8` | Client VM が送ったときのまま |
| 後半（帰り） | `src=8.8.8.8 dst=192.168.22.129` | 前半をそのまま逆にした形（`dst=10.0.0.10`）と**違う** → NAT されている |
| `id=14` | | 前半と後半で同じ。id はずらしていない |

**7行のうち、後半が前半の逆と違うのは ping の行だけ**。NAT されているのは、Client VM からインターネットへの ping だけである。

そのほかの6行は、どれも NAT されていない。

| 行 | 前半（行き） | 何の通信か |
|---|---|---|
| 1, 6, 7 | `192.168.22.1:5361x` → `192.168.22.129:22` | Windows から Router VM への SSH。開いているターミナルの数だけある |
| 2 | `10.0.0.1:50372` → `10.0.0.10:22` | Router VM から Client VM への SSH（`ssh debian-client` の多段 SSH で、Router VM が Client VM へつないでいる部分） |
| 3 | `192.168.22.129:68` → `192.168.22.254:67`（UDP） | DHCP のリース延長。DHCP サーバーへ直接問い合わせている（[docs/concepts/dhcp.md](../../docs/concepts/dhcp.md) の「2 回目以降の問い合わせ」） |
| 4 | `192.168.22.129:22` → `192.168.22.1:57752` | Windows からの SSH だが、前半と後半が逆向き（下の「予想していなかったこと」） |

- SSH の相手 `192.168.22.1` は、VMware が Windows に作った仮想 NIC（VMnet8）の IP。`ProxyJump win` で経由している Windows が、このアドレスでつないできている
- 2行目と3行目は、転送ではなく **Router VM 自身の通信**。conntrack は、転送する通信だけでなく、Router VM 自身の通信や Router VM 宛ての通信も記録する
- TCP の `ESTABLISHED` の記録は、5日間（432,000 秒）通信が無いと消える。ICMP の30秒よりずっと長い。`[ASSURED]` は、両方向にやりとりがあった確かな通信という印

#### 予想していなかったこと：4行目だけ向きが逆

Windows から Router VM への SSH なのに、4行目だけ前半が「`192.168.22.129:22` → `192.168.22.1:57752`」と、Router VM 側から始まった形になっている。

おそらく、**conntrack が動き始める前から開いていた SSH** である。conntrack は `masquerade` の rule を入れた時点で動き始めた（Experiment 5）。それより前から開いていた SSH は、conntrack から見ると途中から見えた通信で、最初に見えたパケット（Router VM から Windows への返事）を「行き」として記録したと考えられる。1, 6, 7 行目が正しい向きなのは、rule を入れた後に tcpdump 用のターミナルなどを新しく開いたからと考えられる。残り秒数が 299 と短い理由は、まだ確かめていない。

### 8. traceroute で経路を見る（Client VM）

labs/03 で予想した「NAT を設定すれば、2台目から先も見える」が確かめられた。

| 何台目 | 返事をした機器 | 何か |
|---|---|---|
| 1 | `10.0.0.1` | Router VM（labs/03 でも見えていた） |
| 2 | `192.168.22.2` | VMware NAT。NAT が無いときは見えなかった |
| 3 | `192.168.11.1` | 家のルーター（家の LAN 側の IP と考えられる） |
| 4〜7 | `* * *` | 返事が無い |
| 8 | `8.8.8.8` | 目的地に届いた |

```text
 Client VM ─→ ①Router VM ─→ ②VMware NAT ─→ ③家のルーター ─→ ④〜⑦プロバイダなど ─→ ⑧8.8.8.8
              10.0.0.1      192.168.22.2    192.168.11.1      （返事なし）
```

- **2台目から先が見えるようになった理由**：NAT が無いときは、2台目に届く時点で送信元が `10.0.0.10` だったので、Time to live exceeded は戻ってこなかった。今は送信元が `192.168.22.129` なので、エラーは Router VM まで戻り、conntrack の記録で `10.0.0.10` へ書き戻される
- Time to live exceeded は、echo reply とは違う種類の ICMP（エラー）。それでも書き戻せるのは、conntrack が「記録済みの通信に関係するエラー」（related）も見分けられるから。ICMP のエラーの中には、原因になった元のパケットのヘッダーが入っていて、conntrack はそれを読んでどの通信のエラーかを判断する
- **2台目・3台目の一部の `*`**：同じ機器でも、3回のうち返事が来ない回があった。ルーターがエラーを返す回数を制限しているためと考えられる
- **4〜7台目の `* * *`**：プロバイダなどのルーターが、Time to live exceeded を返さない設定になっていると考えられる。8台目で 8.8.8.8 に届いているので、経路が途切れているわけではない
- **Windows が1台として出てこない**：VMware NAT は Windows の上で動いているプログラムなので、Windows と VMware NAT がまとめて1台（`192.168.22.2`）に見えていると考えられる。Experiment 6 で見た「帰りの TTL が 128」（VMware NAT が返事を作り直している）とも合っている

### 9. 永続化する：初期状態の `/etc/nftables.conf` を確認する（Router VM）

| 行 | 意味 |
|---|---|
| `#!/usr/sbin/nft -f` | このファイルを直接実行したときに、`nft -f` で読み込むという印 |
| `flush ruleset` | 今カーネルに入っているルールを全部消してから、この後の内容を読み込む |
| `table inet filter { … }` | ファイアウォール用の table（IPv4 と IPv6 の両方）。3つの関所に `filter` の chain があるが、**rule が1つも無い** |

- chain に `policy` が書かれていないので、`policy accept`（そのまま通す）になる。この table は**何もしない**、ファイアウォールのひな形（labs/05 で rule を書き足す）
- **NAT の table は入っていない**。このまま `nftables.service` を有効にしても NAT は効かず、`flush ruleset` で、コマンドで入れた NAT の設定が消える

### 10. 永続化する：`/etc/nftables.conf` に NAT を書き、サービスを有効にする（Router VM）

- `sudo nft -c -f` は何も表示しなかった。書き方は正しい（`-c` は確かめるだけで、カーネルには渡さない）
- `systemctl enable --now` で、`disabled` → `enabled`、`inactive (dead)` → `active (exited)` になった。`exited` は、ルールを渡して正常に仕事を終えたという意味
- `ExecStart=/usr/sbin/nft -f /etc/nftables.conf` が、このサービスが実際にやっていること。**手で打てる `nft -f` で、ファイルを1回読み込むだけ**のサービスである
- `nft list ruleset` に、`table inet filter` と `table ip nat` の2つが出た。ファイルで `policy` を書かなかった chain には `policy accept;` と表示された。書かないと `accept` になる

### 11. 再起動しても NAT が効くか確かめる（Router VM / Client VM）

**再起動した後も、Client VM から 8.8.8.8 へ ping が通った。** NAT の永続化ができた。

ping が通るには、次の2つが両方効いている必要がある。

| 仕組み | 永続化した場所 | 再起動後 |
|---|---|---|
| IP フォワード | `/etc/sysctl.d/`（labs/02） | 効いている（転送されないと、ping は外へ出ない） |
| NAT | `/etc/nftables.conf` + `nftables.service`（Experiment 10） | 効いている（書き換えられないと、返事は戻らない） |
