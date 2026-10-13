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

まとまった説明：[interface.md](../concepts/interface.md)

| 疑問 | 日付 |
|---|---|
| [プライベートIPとグローバルIP：知られると危ないのはどちらか](2026-10-08-private-ip-vs-global-ip.md) | 2026-10-08 |
| [NIC 名（`ens33`）と、1 つの NIC に付ける IP アドレス（数・ネットワーク）](2026-10-11-nic-name-and-multiple-ip-addresses.md) | 2026-10-11 |
| [`ip addr del` でアドレスを消すと、何が消えるのか](2026-10-11-what-ip-addr-del-removes.md) | 2026-10-11 |
| [ルーティングテーブルの `10.0.0.0` とは何か：ネットワークアドレスと `.0` の意味](2026-10-13-network-address-in-routing-table.md) | 2026-10-13 |

## DHCP

まとまった説明：[dhcp.md](../concepts/dhcp.md)

| 疑問 | 日付 |
|---|---|
| [DHCP サーバーのアドレスはどこで分かるのか](2026-10-11-where-to-find-dhcp-server-address.md) | 2026-10-11 |
| [DHCP はルーターの機能か、DHCP サーバーはどこに・いくつ置くのか](2026-10-11-dhcp-role-and-placement.md) | 2026-10-11 |
| [DHCP サーバーはどう作るのか：仕組みと実装方針](2026-10-12-how-to-implement-dhcp-server.md) | 2026-10-12 |
| [DHCP サーバーはどこで動いているのか：VMware・Debian のサービス・実機](2026-10-12-where-dhcp-server-runs.md) | 2026-10-12 |

## セキュリティ

| 疑問 | 日付 |
|---|---|
| [WANの入口は攻撃者の標的になるのか](2026-10-08-is-wan-entrance-a-target.md) | 2026-10-08 |
| [IPアドレスを知られると侵入されるのか（Tailscale・VMのIP）](2026-10-08-is-knowing-ip-address-dangerous.md) | 2026-10-08 |
