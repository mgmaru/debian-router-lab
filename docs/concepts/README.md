# Concepts

ネットワークの概念を整理する。
学習中に出た疑問と、AI に聞いた回答は [docs/questions](../questions/) にある。

## ドキュメント一覧

基礎から順に並べている。

| ドキュメント | 内容 |
|---|---|
| [IP Address](ip-address.md) | ネットワーク部とホスト部、`/24`、サブネットマスク（ネットワークアドレスの暗算を含む）、ネットワークアドレス、ブロードキャストアドレス、機器に付けられるアドレス |
| [Network Interface](interface.md) | `ip a` の読み方（インターフェース名、MAC アドレス、`dynamic` など）、リンクローカルアドレス（`169.254.x.x`） |
| [Routing](routing.md) | デフォルトゲートウェイ：なぜ必要か、設定のしかた、Router VM に `gateway` を書かなかった理由 |
| [Neighbor Table（近隣テーブル）](neighbor.md) | 近隣テーブルと ARP、ルーティングテーブルとの役割分担、`ip neigh` の読み方 |
| [NAT](nat.md) | NAT の本質（出ていくときに送信元を書き換える）、NAT が無いと返事が戻らない理由（tcpdump の結果つき）、行きの書き換えと帰りの書き戻し（conntrack）、IP フォワードとの役割の違い、何段も重なる NAT |
| [ファイアウォール](firewall.md) | rule の評価のしかた（上から順、最初に当たった rule で決まる、条件と動作、終わる動作と続ける動作、順番が結果を変える例、policy、drop と reject の違い）、デフォルト拒否とデフォルト許可の違いとデフォルト拒否が選ばれる理由、forward のデフォルト拒否の形、input をデフォルト拒否にするときの注意 |
| [DHCP](dhcp.md) | DHCP が配るもの、やりとり（DORA）と、1 回目・2 回目以降の問い合わせの宛先、受け取った内容の確認方法、静的 IP との使い分け |
| [一時的な設定と永続的な設定](persistent-config.md) | `ip` コマンドなどの一時的な設定と、`/etc/network/interfaces` などの永続的な設定の違い、設定ごとの永続化の場所、`/etc/sysctl.d/` の書き方と読まれる順番、`/etc` と `/usr` の使い分け、調べ方 |

## Linux カーネル

ルーターの働きを実際に処理している、Linux カーネルの中の仕組みは [kernel/](kernel/README.md) にまとめている。上の一覧には、どのルーターにもあるネットワークの一般的な概念を置いている。基礎から順に並べている。各ドキュメントの詳しい内容は [kernel/README.md](kernel/README.md) を参照。

| ドキュメント | 内容 |
|---|---|
| [Linux カーネル](kernel/README.md) | カーネルとは何か、カーネル空間とユーザー空間、カーネルの動き方を決める3種類のもの、kernel/ に置くものの分け方 |
| [パケットの処理の流れ](kernel/packet-path.md) | NIC に届いてから出ていくまでの処理の順番、sk_buff と下調べの結果、関所に来る前に捨てられるパケット |
| [関所（Netfilter のフック）](kernel/netfilter-hooks.md) | 5つの関所、パケットが通る3つの道、関所ごとに置くものと置かないもの |
| [conntrack（コネクショントラッキング）](kernel/conntrack.md) | 通信の記録の表、2つの向き、返事の見つけ方と NAT の書き戻し |
| [nftables](kernel/nftables.md) | ルールの渡し方、関所の正体と確かめ方、table・chain・rule、マスカレードの入れ方と永続化、ほかのツールとの関係 |
