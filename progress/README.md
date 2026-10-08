# Progress

現在地と次にやることを記録する。

フェーズごとのチェックリストは [todo.md](todo.md) を参照。

## 現在地

現在、VMnet2上で以下の一時設定まで完了している。

```text
Router LAN : 10.0.0.1/24
Client     : 10.0.0.10/24
```

Router VMとClient VMの相互Pingも成功済み。

## 次にやること

次回はまず、

> **Router / ClientのIP設定を永続化する**

ことから進める。

候補はDebianの、

```text
/etc/network/interfaces
```

への設定記述。

永続化後にVMを再起動し、

```bash
ip -br addr
ip route
ping
```

で設定が保持されていることを確認する。

その後、

1. `ip route`
2. `ip neigh`
3. `tcpdump`
4. ClientのDefault Gateway設定
5. IP Forwarding
6. NAT

の順で、本格的なルーター学習へ進む。
