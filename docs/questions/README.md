# Questions

学習中に出た疑問と、AI に聞いた回答をまとめる。
整理済みの説明は [docs/concepts](../concepts/) にある。

## ルーティング・デフォルトゲートウェイ

まとまった説明：[routing.md](../concepts/routing.md)

| 疑問 | 日付 |
|---|---|
| [デフォルトゲートウェイとは「同じネットワークで転送先が見つからない場合の送り先」か](2026-10-08-default-gateway-meaning.md) | 2026-10-08 |
| [`ip addr` にデフォルトゲートウェイは表示されないのか](2026-10-11-ip-addr-and-default-gateway.md) | 2026-10-11 |
| [`ip route` とルーティングテーブルとは何か](2026-10-11-what-is-ip-route-and-routing-table.md) | 2026-10-11 |
| [`192.168.22.2` は何か：VMware NAT とデフォルトゲートウェイ](2026-10-11-what-is-vmware-nat-gateway.md) | 2026-10-11 |

## IP アドレス・NIC

まとまった説明：[ip-address.md](../concepts/ip-address.md)、[interface.md](../concepts/interface.md)

| 疑問 | 日付 |
|---|---|
| [プライベートIPとグローバルIP：知られると危ないのはどちらか](2026-10-08-private-ip-vs-global-ip.md) | 2026-10-08 |
| [NIC 名（`ens33`）と、1 つの NIC に付ける IP アドレス（数・ネットワーク）](2026-10-11-nic-name-and-multiple-ip-addresses.md) | 2026-10-11 |
| [`ip addr del` でアドレスを消すと、何が消えるのか](2026-10-11-what-ip-addr-del-removes.md) | 2026-10-11 |
| [ネットワークアドレスとブロードキャストアドレス：`.0` と `.255` の意味と使い道](2026-10-13-network-address-in-routing-table.md) | 2026-10-13 |
| [`/23` や `/15` のような中途半端な長さのネットワークは、実務で使うのか](2026-10-14-non-octet-prefix-in-practice.md) | 2026-10-14 |
| [`ip link` の `BROADCAST,MULTICAST` は、ブロードキャストやマルチキャストを受け付けるという意味か](2026-10-24-broadcast-multicast-flags-in-ip-link.md) | 2026-10-24 |

## プロトコル（ICMP・TCP・UDP）

| 疑問 | 日付 |
|---|---|
| [ICMP と TCP・UDP の違い：ping にポートやサーバーはあるのか、ping が通るのに TCP が通らないのはなぜか](2026-10-24-icmp-vs-tcp-udp-and-ping.md) | 2026-10-24 |

## IP Forwarding・ルーターの仕組み

| 疑問 | 日付 |
|---|---|
| [IP Forwarding はルーターの機能か：初期状態、専用ルーターとの違い、Docker が転送をオンにする理由](2026-10-24-ip-forwarding-default-and-docker.md) | 2026-10-24 |
| [専用ルーターはなぜ仕事を 2 つに分けるのか：CPU と ASIC の違い](2026-10-24-router-cpu-vs-asic.md) | 2026-10-24 |

## DHCP

まとまった説明：[dhcp.md](../concepts/dhcp.md)

| 疑問 | 日付 |
|---|---|
| [DHCP サーバーのアドレスはどこで分かるのか](2026-10-11-where-to-find-dhcp-server-address.md) | 2026-10-11 |
| [DHCP はルーターの機能か、DHCP サーバーはどこに・いくつ置くのか](2026-10-11-dhcp-role-and-placement.md) | 2026-10-11 |
| [DHCP サーバーはどう作るのか：仕組みと実装方針](2026-10-12-how-to-implement-dhcp-server.md) | 2026-10-12 |
| [DHCP サーバーはどこで動いているのか：VMware・Debian のサービス・実機](2026-10-12-where-dhcp-server-runs.md) | 2026-10-12 |
| [DHCP はいつ、どこ宛てに問い合わせるのか：`255.255.255.255` とサーバーへの直接の問い合わせ](2026-10-14-dhcp-request-destination.md) | 2026-10-14 |

## セキュリティ

| 疑問 | 日付 |
|---|---|
| [WANの入口は攻撃者の標的になるのか](2026-10-08-is-wan-entrance-a-target.md) | 2026-10-08 |
| [IPアドレスを知られると侵入されるのか（Tailscale・VMのIP）](2026-10-08-is-knowing-ip-address-dangerous.md) | 2026-10-08 |
