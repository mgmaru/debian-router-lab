# Deploy

## VMから実機への移植

VMと実機で基本構造は変わらない。

### VM

```text
ens33 → WAN
ens37 → LAN
```

### 実機

```text
eth0 → WAN
eth1 → LAN
```

主な違いは、

- Interface名
- NIC Driver
- MAC Address
- CPU Architecture
- Hardware性能

程度である。

ソフトウェアをハードウェア依存にしなければ、

```text
Debian VM
   ↓
自作Router Software
   ↓
実機Debian
```

へ比較的容易に移植できる。
