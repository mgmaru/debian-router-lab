# VMware

## VMware Workstation Pro

Windows 11へ VMware Workstation Pro 26H1u1 をインストールした。

VMware Workstation Pro上に、以下2台のVMを構築した。

```text
VM 1: debian-router
VM 2: debian-client
```

仮想ネットワークの構成は [network-topology.md](network-topology.md)、Debianのインストール設定は [environment/debian](../debian/README.md) を参照。

---

## debian-router

| 項目 | 設定 |
|---|---|
| VM名 | `debian-router` |
| OS | Debian 13 |
| Architecture | amd64 |
| CPU | 2 vCPU |
| RAM | 2 GB |
| Disk | 20 GB |
| Disk形式 | Single file |
| 初期NIC | NAT |

---

## debian-client

| 項目 | 設定 |
|---|---|
| VM名 | `debian-client` |
| OS | Debian 13 |
| Architecture | amd64 |
| CPU | 1 vCPU |
| RAM | 1 GB |
| Disk | 10 GB |
| Disk形式 | Single file |
| NIC | 1枚 |
| インストール時NIC | NAT |
