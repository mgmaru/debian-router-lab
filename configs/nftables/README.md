# nftables

nftables のルール（NAT・ファイアウォール）を永続化するために、`/etc/nftables.conf` に配置する設定ファイル。

| ファイル | 配置先 | 内容 |
|---|---|---|
| [nftables.conf](nftables.conf) | debian-router の `/etc/nftables.conf` | ens33（WAN側）から出ていくパケットのマスカレード（`table ip nat`）。`table inet filter` は Debian の初期状態のまま（rule が無く、すべて通す） |

起動時に読み込むには、`nftables` サービスを有効にする（Debian の初期状態では `disabled`）。

```bash
sudo nft -c -f /etc/nftables.conf      # 書き方だけを確かめる
sudo systemctl enable --now nftables   # 起動時に読み込むようにし、今すぐ読み込む
```

反映の手順は [labs/04-nat](../../labs/04-nat/README.md) の Experiment 9〜11、ルールの書き方は [docs/concepts/kernel/nftables.md](../../docs/concepts/kernel/nftables.md) と [docs/tools/nft.md](../../docs/tools/nft.md) を参照。

`table inet filter` は、labs/05 でファイアウォールの rule を書き足す。
