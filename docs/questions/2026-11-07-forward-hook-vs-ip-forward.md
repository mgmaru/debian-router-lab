# 関所の forward と ip_forward は違うのか：設定値と関所の違い、通路と検査場、両方が要る理由

調査日：2026-11-07

関連：

- [Linux カーネル](../concepts/kernel/README.md)（カーネルの動き方を決める3種類のもの：設定値と、参照するルール）
- [パケットの処理の流れ](../concepts/kernel/packet-path.md)（ルーティング判断で `ip_forward` を確かめること、関所に来る前に捨てられるパケット）
- [関所（Netfilter のフック）](../concepts/kernel/netfilter-hooks.md)
- [IP Forwarding はルーターの機能か](2026-10-24-ip-forwarding-default-and-docker.md)
- [labs/02-ip-forwarding](../../labs/02-ip-forwarding/README.md)、[labs/05-firewall](../../labs/05-firewall/README.md)

## 疑問

> **質問81**：関所の forward は、ルーターの転送機能の `ip_forward` とは違うのか？

## 結論

**違うもの。** 名前が似ているが、役割が違う。

| | `ip_forward` | 関所の forward |
|---|---|---|
| 正体 | **設定値**（スイッチ）。`net.ipv4.ip_forward` | **関所**（パケットの通り道にある呼び出し口） |
| 決めること | **転送するかどうか**（全部まとめて） | 転送するパケットを、**1つずつ** rule で調べる場所 |
| 値・中身 | 0 か 1 | 置いた chain の rule（無ければ素通り） |
| 変えるコマンド | `sysctl` | `nft` |
| カーネルの動き方を決める3種類のうち | 設定値 | 参照するルールを置く場所 |

`ip_forward` は「**通路を開けるかどうか**」、関所の forward は「**その通路の途中にある検査場**」。ルーターとして転送しつつファイアウォールで選ぶには、両方が要る。

## 2つの関係：通路と検査場

```text
 パケットが届く → ① prerouting → ルーティング判断
                                     │
                                     ├─ 自分宛て → ② input
                                     │
                                     └─ 転送 ── ip_forward は？
                                                  │
                                                  ├─ 0：通路が閉じている → 捨てる（forward まで来ない）
                                                  │
                                                  └─ 1：通路が開いている
                                                         │
                                                         ▼
                                                     ③ forward（検査場）── rule で通す・捨てる
                                                         │
                                                         ▼
                                                     ⑤ postrouting → 出ていく
```

- `ip_forward` が確かめられるのは**ルーティング判断**のとき。ここで 0 なら、パケットは ③ forward に来る前に捨てられる
- `ip_forward` が 1 なら、転送するパケットは ③ forward を通る。forward に rule が無ければ素通りする

### 組み合わせと結果

| ip_forward | forward の rule | 結果 |
|---|---|---|
| 0 | 何があっても | **すべて転送されない**。パケットは forward まで来ない |
| 1 | 無い（今の Router VM） | **すべて転送される**（素通り） |
| 1 | デフォルト拒否＋許可の rule（labs/05 の実験 2） | **許可したものだけ転送される** |

ルーターとして転送しつつ、ファイアウォールで選ぶには、`ip_forward = 1` で通路を開けてから、forward の rule で何を通すかを選ぶ。

## 似た結果になる場合

forward に `policy drop` だけを付けて rule を1つも書かないと、`ip_forward = 0` と同じように何も転送されなくなる。ただし、止まる場所と仕組みが違う。

| | `ip_forward = 0` | forward に `policy drop` だけ |
|---|---|---|
| 止まる場所 | ルーティング判断 | ③ forward |
| 一部だけ通す | できない（0 か 1 だけ） | **できる**（許可の rule を足す） |
| `counter` で数えられるか | 数えられない（関所に来ない） | 数えられる |

[パケットの処理の流れ](../concepts/kernel/packet-path.md) の「関所に来る前に捨てられるパケット」の表にある「`ip_forward` が 0」の行が、左側の場合にあたる。

## 名前が似ている理由

どちらも「転送（forward）」に関わるものだから。ただし、`ip_forward` は「転送する機能そのものの ON/OFF」、関所の forward は「転送されるパケットが通る場所の名前」で、指しているものが違う。

| 名前 | 指しているもの |
|---|---|
| IP フォワード、`ip_forward` | 転送する**機能**と、その ON/OFF の設定値 |
| forward（関所） | 転送されるパケットが通る**場所** |
| forward（chain の名前） | その場所に置いた chain に、自分で付けたラベル（[netfilter-hooks.md](../concepts/kernel/netfilter-hooks.md) の「このラボで置くもの」） |
