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
