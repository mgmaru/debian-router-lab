# 02 - IP Forwarding

## Goal

Linuxは初期状態では通常、別NIC間のパケット転送を行わない。

IP Forwardingを有効化し、

```text
WAN
 ↓
Linux
 ↓
LAN
```

の中継を可能にする。

学習項目：

- RouterとHostの違い
- IP Forwarding
- Linux KernelのPacket Forwarding

## Notes

Linux は既定では、自分宛てでないパケットを受け取っても捨てる。これを中継するように変える。

```bash
# 一時的
sudo sysctl -w net.ipv4.ip_forward=1

# 永続化（起動時に読まれる場所に書く）
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-router.conf
sudo sysctl --system
```

> 💡 Debian 13 では `/etc/sysctl.conf` が標準では存在しないので、`/etc/sysctl.d/` に書くのが確実。

永続化の考え方は [persistent-config.md](../../docs/concepts/persistent-config.md) を参照。
