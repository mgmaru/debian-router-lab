# 005. VMへのSSHは踏み台経由で行う

## Status

Accepted

## Context

VMwareのコンソールは操作しにくいため、普段の操作はSSHを使用したい。

また、外出先のMacBookからもRouter VMとClient VMを操作したい。

## Decision

VMへのSSHは、手前のマシンを踏み台にした多段SSHで行う。

```text
MacBook
   │
   │ Tailscale
   ▼
Windows
   │
   │ SSH ProxyJump
   ▼
debian-router
   │
   │ SSH ProxyJump（VMnet2）
   ▼
debian-client
```

- Router VMへTailscaleを直接入れず、Windowsを踏み台にする
- Client VMには管理用NICを追加せず、Router VMを踏み台にする

## Reasons

### Router VMにTailscaleを入れない理由

Router VMへTailscaleを直接入れず、Windowsを踏み台にすることで、Router VMのネットワーク構成をシンプルに保つ。

### Client VMに管理用NICを追加しない理由

ClientにVMnet8などの管理用NICを追加すると、

```text
Client
 ├─ LAN → Router
 └─ NAT → Internet
```

のようにRouterを迂回する経路ができてしまい、学習用ネットワークが分かりにくくなる。

そのため、

> **ClientはLAN用NIC 1枚のみ**

とし、

> **ClientへのSSHはRouterを踏み台にして行う**

ことで、Clientへの管理通信もRouterを経由させ、学習用ネットワーク構成を維持する。

## Consequences

Router VMへの管理用SSH通信はWAN側（VMnet8）を通り、Router / Client間の学習用通信はLAN側（VMnet2）を通る。

Firewall学習時にWAN側のTCP/22を誤って遮断するとSSHできなくなる可能性があるため、VMware Consoleは復旧手段として残しておく。

具体的な接続方法は [environment/remote-access](../../environment/remote-access/README.md) を参照。
