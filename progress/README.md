# Progress

現在地と次にやることを記録する。

フェーズごとのチェックリストは [todo.md](todo.md) を参照。

## 現在地

Router / ClientのIP設定を `/etc/network/interfaces` に書き、永続化した（[labs/01-network-interface](../labs/01-network-interface/README.md)）。

```text
Router WAN : 192.168.22.129/24（ens33、DHCP）
Router LAN : 10.0.0.1/24（ens37、静的）
Client     : 10.0.0.10/24（ens33、静的、ゲートウェイ 10.0.0.1）
```

現時点でできていること・いないこと：

| 通信 | 状態 | 理由 |
|---|---|---|
| クライアント VM → ルーター VM（10.0.0.1） | ✅ できる | 同じネットワーク内 |
| クライアント VM → ルーター VM の外側の IP | ✅ できる | ルーター VM 自身のアドレス宛てなので、ルーター VM が直接返事をする |
| クライアント VM → インターネット | ❌ まだできない | ルーター VM が「受け取った通信を中継する」設定をしていない |

## 次にやること

ルーターとして機能させるため、次の順で進める。

1. `ip route` / `ip neigh` / `tcpdump` を理解する
2. IP Forwarding（[labs/02-ip-forwarding](../labs/02-ip-forwarding/README.md)）
3. NAT（[labs/04-nat](../labs/04-nat/README.md)）
4. ClientのDNS設定（[labs/07-dns](../labs/07-dns/README.md)）
5. DHCPサーバー（任意、[labs/06-dhcp](../labs/06-dhcp/README.md)）
