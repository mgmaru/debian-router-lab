# 01 - Network Interface

## Goal

まずNICとIPアドレスを理解する。

主に使用するもの：

```bash
ip addr
ip link
ip route
```

学習項目：

- NIC
- MAC Address
- IP Address
- Subnet
- Default Gateway

## Topology

```text
Debian Router VM
LAN: 10.0.0.1/24

Debian Client VM
LAN: 10.0.0.10/24
```

```text
Router 10.0.0.1
    │
    │ VMnet2
    │
Client 10.0.0.10
```

## Experiment

Router VMとClient VMをVMnet2へ接続し、LAN側IPを一時的に設定した。

一時設定には `ip` コマンドを使用した。

Router側の例：

```bash
sudo ip link set <LAN_NIC> up
sudo ip addr add 10.0.0.1/24 dev <LAN_NIC>
```

Client側の例：

```bash
sudo ip link set <CLIENT_NIC> up
sudo ip addr add 10.0.0.10/24 dev <CLIENT_NIC>
```

この状態で、Router VMとClient VMの相互通信を確認した。

ClientからRouter：

```bash
ping 10.0.0.1
```

RouterからClient：

```bash
ping 10.0.0.10
```

## Result

どちらも成功し、**VMnet2上でRouter VMとClient VMが相互通信できることを確認済み**。

### 現在の注意点

今回のIP設定は、

```bash
ip addr add ...
```

による一時設定である。

そのため、

```text
一時設定
   ↓
VM再起動
   ↓
IP設定が消える
```

という状態。

永続化は次回作業として、

```text
/etc/network/interfaces
```

などのDebianのネットワーク設定へ記述する予定。

現時点では、

```text
一時設定で疎通確認
        ↓
仕組みを理解
        ↓
永続設定へ移行
```

という順序で進めている。
