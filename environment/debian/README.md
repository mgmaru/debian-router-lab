# Debian

VMのスペックは [environment/vmware](../vmware/README.md) を参照。

IPアドレスの永続化（`/etc/network/interfaces`）の手順は [network.md](network.md) を参照。

## debian-router

使用したISO：

```text
debian-13.7.0-amd64-netinst.iso
```

### インストール設定

| 項目 | 設定 |
|---|---|
| Language | English |
| Location | Japan |
| Locale | en_US.UTF-8 |
| Keyboard | American English |
| Hostname | `debian-router` |
| Root password | 未設定 |
| User | `kajih` |
| Partition | Guided - use entire disk |
| Layout | All files in one partition |
| Package survey | No |
| SSH server | Install |
| Standard system utilities | Install |
| Desktop environment | Installしない |
| GRUB | Install |

---

## debian-client

Client VMもGUIなしの最小Debian構成とした。

主な用途：

```text
ping
curl
traceroute
dig
```

などを使ってRouter VMへ通信を発生させる。
