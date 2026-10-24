# sysctl

カーネルの設定（カーネルパラメータ）を永続化するために、`/etc/sysctl.d/` に配置する設定ファイル。

| ファイル | 配置先 | 内容 |
|---|---|---|
| [99-router.conf](99-router.conf) | debian-router の `/etc/sysctl.d/99-router.conf` | IPv4 の転送を有効にする（`net.ipv4.ip_forward = 1`） |

反映の手順は [labs/02-ip-forwarding](../../labs/02-ip-forwarding/README.md#5-ip-forwarding-の永続化router-vm)、ファイル名・書式・読まれる順番は [persistent-config.md](../../docs/concepts/persistent-config.md#カーネルの設定etcsysctld) を参照。
