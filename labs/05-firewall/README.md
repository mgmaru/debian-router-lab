# 05 - Firewall

## Goal

通信の許可・拒否を制御する。

例：

```text
LAN → Internet : Allow
Internet → LAN : Deny
SSH → Router   : 条件付きAllow
```

利用：

```text
nftables
```

## Notes

Router VMへの管理用SSHはWAN側NIC（VMnet8）を通っている。

WAN側のTCP/22を誤って遮断するとSSHできなくなるため、VMware Consoleを復旧手段として残しておく（[environment/remote-access](../../environment/remote-access/README.md) を参照）。
