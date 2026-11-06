# policy は必ず使われるのか：書く場所と使われるタイミング、rule で決まったパケットとの関係、policy は強いルールか

調査日：2026-11-06

関連：

- [ファイアウォール](../concepts/firewall.md)（rule の評価のしかた、policy、デフォルト拒否の考え方）
- [rule の順番で結果はどう変わるのか](2026-11-06-rule-order-changes-result.md)
- [labs/05-firewall](../../labs/05-firewall/README.md)

## 疑問

> **質問75**：firewall.md の「デフォルト拒否の考え方」を読んでいて疑問に思った。policy は最後に書かれていても、必ず読まれるのか？ それとも、`accept` や `drop` によって読み飛ばされるのか？

> **質問76（追加質問）**：policy が使われるのは、最初に書いても、rule を読んだ後の最後ということか？

> **質問77（追加の考え）**：policy は「どの rule にも決めてもらえなかったパケットの行き先」と考えるのが良さそうだ。個人的には強いルールだと思っていたが、rule が先に読まれて適用される時点で、強いとは言えない。

## 結論

| 質問 | 答え |
|---|---|
| policy は必ず使われる？ | **いいえ。** どの rule でも `accept`・`drop`・`reject` が決まらなかったパケットにだけ使われる |
| `accept` や `drop` で読み飛ばされる？ | **はい。** rule で結果が決まったパケットは、そこで chain を抜けるので、policy は使われない |
| policy を書くのは最初で、使われるのは最後？ | **そのとおり。** 書くのは chain の先頭（type の行）。使われるのは、rule をすべて調べ終わって、どれでも結果が決まらなかったとき |
| policy は強いルールか？ | **優先される順番では、いちばん弱い**（最後に回される）。ただし**影響する範囲はいちばん広い**（rule に書かれていないパケットすべて） |

policy は「最後に必ず通るもの」ではなく、**「どの rule にも決めてもらえなかったパケットの行き先」**と考える。

## 質問75：policy は必ず使われるのか

### policy は「最後の行」ではない

policy は chain の最後の行ではなく、**chain の先頭（type の行）**に書く。

```text
chain forward {
	type filter hook forward priority filter; policy drop;   ← policy はここに書く
	ct state established,related accept                      ← rule 1
	iifname "ens37" oifname "ens33" accept                   ← rule 2
}
```

firewall.md の「chain は上から順に読むチェックリスト」の図で policy を一番下に描いているのは、**使われるタイミング**を表すため。書く場所と使われるタイミングが違うので、紛らわしい。

### パケットごとに見ると

| パケット | rule 1 | rule 2 | policy | 結果 |
|---|---|---|---|---|
| 8.8.8.8 → Client VM（返事） | **accept** で決まる | 調べない | **使われない** | 通る |
| Client VM → 8.8.8.8（最初の行き） | 当たらない | **accept** で決まる | **使われない** | 通る |
| WAN の機器 → 10.0.0.10:22（新しい接続） | 当たらない | 当たらない | **drop が使われる** | 捨てる |

`accept` や `drop` で決まったパケットは、そこで chain を抜ける。policy まで行かない。

### 「続ける」動作だけなら policy まで行く

`counter` や `log` は結果を決めない動作なので、それだけの rule に当たっても次へ進む。

```text
chain input {
	type filter hook input priority filter; policy drop;
	tcp dport 22 counter        ← 数えるだけ。accept が無い
}
```

この chain では、SSH のパケットは rule に当たって数えられる。しかし結果は決まらないので、policy の `drop` で捨てられる。SSH を通したいなら、`tcp dport 22 counter accept` のように `accept` まで書く。

## 質問76：書くのは最初、使われるのは最後

そのとおり。条件を付けると、**rule をすべて調べ終わって、どの rule でも結果（`accept`・`drop`・`reject`）が決まらなかった場合だけ**、最後に policy が使われる。

```text
chain forward {
	type filter hook forward priority filter; policy drop;   ← 書くのは最初
	rule 1
	rule 2
	                                                          ← 使われるのはここ（どれでも決まらなかったときだけ）
}
```

| 書く場所 | 使われるタイミング | 使われないとき |
|---|---|---|
| 最初（type の行） | rule をすべて調べ終わった、**最後** | 途中の rule で `accept`・`drop`・`reject` が決まったとき |

## 質問77：policy は強いルールか

**優先される順番で言えば、policy はいちばん弱い。** どの rule よりも後に回され、rule が何も決めなかったときにしか出番がない。

ただし、別の見方をすると、policy は**影響する範囲がいちばん広い**。

| 見方 | policy | rule |
|---|---|---|
| 優先される順番 | **いちばん弱い**（最後に回される） | 上にあるものほど先に決める |
| 影響する範囲 | **いちばん広い**（rule に書かれていないパケットすべて） | 条件に当たったパケットだけ |

- デフォルト拒否が「許可していないものはすべて止まる」と言えるのは、この広さのおかげ
- rule が1つも無い chain に `policy drop` を付けると、すべてのパケットが止まる。input でこれをやると SSH で締め出されるのは、このため

policy は「rule を押しのける強さ」は持っていない。しかし、「rule に書き忘れたものすべての行き先を決める」という意味で、ファイアウォール全体の性格（デフォルト許可か、デフォルト拒否か）を決める重要な設定である。
