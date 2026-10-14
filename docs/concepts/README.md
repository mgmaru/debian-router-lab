# Concepts

ネットワークの概念を整理する。
学習中に出た疑問と、AI に聞いた回答は [docs/questions](../questions/) にある。

## ドキュメント一覧

基礎から順に並べている。

| ドキュメント | 内容 |
|---|---|
| [IP Address](ip-address.md) | ネットワーク部とホスト部、`/24` とサブネットマスク、ネットワークアドレス、ブロードキャストアドレス、機器に付けられるアドレス |
| [Network Interface](interface.md) | `ip a` の読み方（インターフェース名、MAC アドレス、`dynamic` など）、リンクローカルアドレス（`169.254.x.x`） |
| [Routing](routing.md) | デフォルトゲートウェイ：なぜ必要か、設定のしかた、Router VM に `gateway` を書かなかった理由 |
| [Neighbor Table（近隣テーブル）](neighbor.md) | 近隣テーブルと ARP、ルーティングテーブルとの役割分担、`ip neigh` の読み方 |
| [DHCP](dhcp.md) | DHCP が配るもの、やりとり（DORA）と、1 回目・2 回目以降の問い合わせの宛先、受け取った内容の確認方法、静的 IP との使い分け |
| [一時的な設定と永続的な設定](persistent-config.md) | `ip` コマンドなどの一時的な設定と、`/etc/network/interfaces` などの永続的な設定の違い |
