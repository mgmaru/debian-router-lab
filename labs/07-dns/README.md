# 07 - DNS

## Goal

名前解決の役割を理解する。

```text
example.com
    ↓
DNS
    ↓
IP Address
```

ルーター自身をDNS Forwarderとして動作させることも検討する。

## Notes

IP で通信できるようになっても、`google.com` のような名前を引くには DNS サーバーの指定（`/etc/resolv.conf`）が必要である。

クライアント VM は静的 IP にしたため、DHCP から DNS サーバーを受け取っていない（[dhcp.md](../../docs/concepts/dhcp.md)）。
