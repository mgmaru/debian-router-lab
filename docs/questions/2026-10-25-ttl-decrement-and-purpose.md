# TTL はいつ減り、何のためにあるのか：64 台を超えるルーターを通ることはないのか、Time to live exceeded とは何か

調査日：2026-10-25

関連：

- [labs/03-routing](../../labs/03-routing/README.md)（Experiment 3：転送で TTL が 64 → 63 になることを観察）
- [IP Forwarding はルーターの機能か：初期状態、専用ルーターとの違い、Docker が転送をオンにする理由](2026-10-24-ip-forwarding-default-and-docker.md)（転送の手順 ①〜⑥）

## 疑問

> **質問55**：TTL が Router VM の ens37 → ens33 へ IP フォワーディングした時点で 1 減っていた。TTL は IP フォワーディングが起こった時点で 1 減るのか？

> **質問56**：TTL は何のためにあるのか？

> **質問57（追加質問）**：ループしないために TTL がある。Linux は 64 から始まるとのことだが、64 台以上のルーターを通る場合はないのか？ 意図せず捨てられてしまうことがある気がする。

> **質問58（追加質問）**：Client VM で `ping -c 1 -t 1 8.8.8.8` を打つと、`From 10.0.0.1 icmp_seq=1 Time to live exceeded` と出た。Time to live exceeded とは何か？

## 結論

| 質問 | 答え |
|---|---|
| TTL は転送した時点で 1 減る？ | **その通り。** Linux では、**転送すると決まったパケットだけ**、送り出す前に 1 減らす。自分宛てのパケットや、自分で送るパケットでは減らない |
| TTL は何のため？ | **経路がループしたときに、パケットが永遠に回り続けないようにするため。** 各ルーターは次の 1 台しか決めておらず、経路全体を誰も見ていないので、パケット自身に「残り寿命」を持たせている |
| Time to live exceeded とは？ | **TTL が尽きたのでパケットを捨てた、とルーターが送信元に知らせる ICMP のエラー**（Type 11、Code 0）。今回は Router VM（`10.0.0.1`）が返した |
| 64 台を超えることはない？ | **実際のインターネットでは、ほとんど無い。** 世界中どこへ送っても、通るルーターはおよそ 10〜30 台。64 は、それに対して余裕を持たせた値である。足りなくなっても黙って消えるわけではなく、ICMP の「Time to live exceeded」が返る |

---

## 質問55：TTL は転送した時点で 1 減るのか

### 転送すると決まってから、送り出す前に減らす

Router VM の中では、次の順番で処理される。

```text
ens37 で受信                     ← tcpdump -i ens37 はここで見ている（ttl 64）
  ↓
ルーティングテーブルを引く（宛先は自分か？ 転送か？）
  ↓ 転送と決まった（ip_forward = 1）
TTL を確認：1 以下なら捨てて、送信元に「Time to live exceeded」を返す
  ↓
TTL を 1 減らす（64 → 63）、IP ヘッダのチェックサムを計算し直す
  ↓
MAC アドレスを付け替える（近隣テーブルで次の相手の MAC を調べる）
  ↓
ens33 から送信                   ← tcpdump -i ens33 はここで見ている（ttl 63）
```

`tcpdump` は NIC を通るところでパケットを見ているので、ens37 では減る前の 64、ens33 では減った後の 63 が見えた。

TTL が変わると IP ヘッダの中身が変わるので、チェックサム（ヘッダが壊れていないかを確かめる値）も計算し直す。`tcpdump` に `-vv` を付けると `cksum` として表示される。

### TTL が減らないパケット

| パケット | TTL | 理由 |
|---|---|---|
| Router VM **宛て**のパケット（Client VM から `10.0.0.1` への `ping` など） | 減らない | 転送しないため |
| `ip_forward = 0` で捨てられるパケット | 減らない | 転送する前に捨てられるため |
| Router VM **自身が送る**パケット | 減らない | 最初の値（Linux は 64）で出ていく |

「ルーターを 1 台通る ＝ 1 減る」と考えてよい。

---

## 質問56：TTL は何のためにあるのか

### ループしたパケットを消すため

ルーティングテーブルの設定を間違えると、次のようなループが起きることがある。

```text
Router A：8.8.8.8 は Router B へ
Router B：8.8.8.8 は Router A へ
→ A → B → A → B → … と回り続ける
```

TTL が無いと、このパケットは消えずに回り続ける。新しいパケットも次々に加わるので、回線がいっぱいになってしまう。TTL があれば、回るたびに 1 ずつ減り、0 になったところで捨てられる。

labs/03 の Result 2 の通り、各ルーターはルーティングテーブルで**次の 1 台**だけを決めていて、経路全体を知っている機器はいない。そのため、ループを止める仕組みが、パケット自身に入っている。

### 補足

- **名前の由来**：もともとは「秒数」で寿命を表す設計だった（Time To Live）。実際には「通ったルーターの数」として使われている。IPv6 では、実態に合わせて **Hop Limit** という名前になった
- **最初の値**：IP ヘッダの 8 ビットの欄なので、最大 255。最初の値は OS によって違い、Linux は 64、Windows は 128
- **traceroute**：TTL をわざと 1、2、3…と小さくして送り、各ルーターが返す「Time to live exceeded」から、経路を 1 台ずつ調べる

---

## 質問57：64 台を超えるルーターを通ることはないのか

### インターネットの経路は思ったより短い

世界中のどこへ送っても、通るルーターはおよそ **10〜30 台** である。インターネットは、ルーターが一列につながっているのではなく、階層になっているからである。

```text
自宅ルーター → プロバイダ → 大手回線事業者（バックボーン） → 相手側のプロバイダ → 相手
```

大手回線事業者どうしは少ない段数で直接つながっていて、遠い国へも海底ケーブルで一気に渡る。

### 64 は「余裕を持たせた値」

64 は、IANA が標準の初期値として推奨している値である（RFC 1700）。実際の経路の長さ（最大でも 30〜40 台程度）に対して、2 倍くらいの余裕がある。

| 初期値 | 起きること |
|---|---|
| 大きすぎる | ループしたパケットが長く回り続け、回線に負担がかかる |
| 小さすぎる | 正常なパケットまで、途中で捨てられる |

### 足りなかったら、どうなるか

意図せず捨てられることは、起こりうる。ただし、TTL が 0 になったルーターは、送信元に ICMP の「Time to live exceeded」を返すので、黙って消えるわけではない。初期値は変更できる（最大 255）。

```bash
# Linux の初期値
sysctl net.ipv4.ip_default_ttl   # 64
```

---

## 質問58：Time to live exceeded とは何か

### TTL が尽きたことを知らせる ICMP のエラー

ルーターは、TTL が 1 以下のパケットを転送できないので捨てる（質問55）。そのとき、送信元に「**捨てました**」と知らせるのが、ICMP の **Time Exceeded（Type 11）、Code 0（転送中に TTL が尽きた）** である。`ping` はこれを `Time to live exceeded` と表示する。

```text
Client VM：TTL=1 で 8.8.8.8 へ送る
   ↓
Router VM：転送すると決まった → TTL を確認 → 1 以下なので転送できない
   ↓ パケットを捨てる
   ↓ 代わりに、送信元（10.0.0.10）へ ICMP「Time to live exceeded」を送る
Client VM：From 10.0.0.1 ... Time to live exceeded と表示
```

### 結果の読み方

```text
From 10.0.0.1 icmp_seq=1 Time to live exceeded
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

| 部分 | 意味 |
|---|---|
| `From 10.0.0.1` | エラーを返したのは Router VM。`8.8.8.8` からの返事ではない |
| `icmp_seq=1` | 1 つ目の echo request についてのエラー。エラーの中に捨てたパケットの先頭（IP ヘッダと、その後ろの 8 バイト）が入っているので、送信元はどれについてのエラーか分かる |
| `0 received, +1 errors` | echo reply は 0 個、エラーが 1 個届いた |
| `100% packet loss` | `8.8.8.8` からの返事は来なかった |

### 黙って捨てる場合との違い

| 場面 | ルーターの動き | 送信元から見えること |
|---|---|---|
| `ip_forward = 0`（labs/02） | 黙って捨てる | 何も返らない |
| TTL が尽きた | 捨てて、**ICMP で知らせる** | `Time to live exceeded` |

このエラーは、NAT が無くても Client VM に届いた。返したのは Router VM 自身で、Client VM と同じ LAN（`10.0.0.0/24`）にいるので、帰り道に困らないからである。

### traceroute はこのエラーを使う

`traceroute` は、TTL を 1、2、3…と増やしながら送り、各ルーターが返す「Time to live exceeded」の送信元から、経路を 1 台ずつ調べる。今は NAT が無いので、2 台目（`192.168.22.2`）から先の返事は Client VM に戻らない。

---

## まだ試していないこと

| コマンド | 予想される結果 | 確かめられること |
|---|---|---|
| （Router VM）`sudo sysctl net.ipv4.ip_default_ttl` | `64` | Linux の初期値（質問57） |
| （MacBook）`ping -c 3 8.8.8.8` | `ttl=` が 100〜120 くらい | 相手の初期値（128 など）から引くと、通ったルーターの台数が推測できる。64 台よりかなり少ないはず（質問57） |
| （Router VM）`sudo tcpdump -i ens33 -n -vv icmp` を動かしたまま、（Client VM）`ping -c 1 8.8.8.8` | ens37 と ens33 で `cksum` の値が違う | TTL を減らすと、チェックサムも計算し直される（質問55） |

---

## まとめ

- Linux は、**転送すると決まったパケットだけ**、送り出す前に TTL を 1 減らす。1 以下なら捨てて、送信元に「Time to live exceeded」を返す。自分宛てのパケットや、自分で送るパケットでは減らない。
- TTL は、**経路がループしたときにパケットを消すための残り寿命**である。経路全体を知っている機器がいないので、パケット自身に寿命を持たせている。
- TTL が尽きると、ルーターは捨てて、送信元に ICMP の「**Time to live exceeded**」を返す。`traceroute` はこれを使って経路を調べる。
- インターネットの経路は **10〜30 台** 程度で、初期値 64 には十分な余裕がある。足りなくなっても、ICMP で知らされる。

---

## 参考

- [RFC 791：Internet Protocol（Time to Live）](https://datatracker.ietf.org/doc/html/rfc791)
- [RFC 792：Internet Control Message Protocol（Time Exceeded Message）](https://datatracker.ietf.org/doc/html/rfc792)
- [RFC 1700：Assigned Numbers（IP TTL の推奨初期値 64）](https://datatracker.ietf.org/doc/html/rfc1700)
- [RFC 1812：Requirements for IP Version 4 Routers（転送するときに TTL を減らす）](https://datatracker.ietf.org/doc/html/rfc1812)
- [Linux カーネル：ip-sysctl（`ip_default_ttl`）](https://docs.kernel.org/networking/ip-sysctl.html)
