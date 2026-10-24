# カーネルの設定の永続化：`/etc/sysctl.d/` の仕組み、ファイル名と書式、読まれる順番、`/etc` と `/usr` の使い分け

調査日：2026-10-25

まとまった説明：[docs/concepts/persistent-config.md](../concepts/persistent-config.md)

関連：

- [labs/02-ip-forwarding](../../labs/02-ip-forwarding/README.md)（`sysctl` と `net.ipv4.ip_forward`）
- [IP Forwarding はルーターの機能か：初期状態、専用ルーターとの違い、Docker が転送をオンにする理由](2026-10-24-ip-forwarding-default-and-docker.md)
- [environment/debian/network.md](../../environment/debian/network.md)（`/etc/network/interfaces` による IP アドレスの永続化）

## 疑問

> **質問50**：カーネルの設定は、`/etc/sysctl.d/` の下にファイルを作ることで永続化できる、という認識で合っているか？

> **質問51**：そのファイルの名前や書式に、決まりはあるか？

> **質問52**：以前、ネットワークインターフェースの設定もファイルを作って永続化した。このような永続化の設定が書かれているサイトはあるか？

> **質問53（追加質問）**：設定が読まれる場所が 4 つほどあった。これらのうち、`/etc` が最優先されると覚えておけばよいか？ 正しくは、`/etc` が最後に読まれる、という感じか？

> **質問54（追加質問）**：`/etc` 以外の場所のファイルには、自分の設定を書き込まないほうがよいのか？

## 結論

| 質問 | 答え |
|---|---|
| `/etc/sysctl.d/` で永続化できる？ | **できる。** 起動時に `systemd-sysctl` というサービスが `/etc/sysctl.d/*.conf` などを読み、書かれた値をカーネルに設定する。ただし、ファイルを作っただけでは今のカーネルには反映されないので、`sudo sysctl --system` で読み直す |
| ファイル名や書式の決まりは？ | **ある。** ファイル名は **`.conf` で終わる**必要があり、**名前の順に読まれる**。書式は **1 行に 1 つ `項目名 = 値`**、`#` か `;` で始まる行はコメント。先頭の数字（`99-` など）は決まりではなく、読む順番を決めるための慣習 |
| 永続化の方法がまとまったサイトは？ | **全部がまとまったサイトは無い。** 設定ごとに、「起動時にその設定を読むソフトウェア」の公式ドキュメント（主に man ページ）に書かれている。`man sysctl.d`、`man interfaces` など。Debian の man ページは <https://manpages.debian.org/> で読める |
| `/etc` が最優先？ 最後に読まれる？ | **どちらも少し違う。** **同じファイル名**なら `/etc` のものだけが使われる（`/etc` が勝つ）。しかし**違うファイル名**なら、場所に関係なく**ファイル名の順**に読まれ、後のものが勝つ。`/etc` に書いても、ファイル名の順番が先なら、`/usr/lib` の設定に負けることがある。だから自分のファイルには `99-` を付ける |
| `/etc` 以外には書かない？ | **書かない。** `/run` は再起動で消え、`/usr/lib` はパッケージの更新で上書きされることがある。「**パッケージ（OS の標準）の設定は `/usr`、管理者（自分）の設定は `/etc`**」という分け方は、sysctl だけでなく Linux 全体の考え方である |

---

## 質問50：`/etc/sysctl.d/` で永続化できるか

### できる

`sysctl -w` で変えた値は、**動いているカーネルの中**にだけある。カーネルは再起動でまっさらになるので、値も初期値に戻る。labs/02 で `ip_forward` を `1` にしたあと、再起動すれば `0` に戻るのはこのためである。

永続化するには、**起動時に読まれる場所**に設定を書く。カーネルの設定の場合、その場所が `/etc/sysctl.d/` である。

### 起動時に何が起きているか

```text
電源オン
   ↓
カーネルが起動する（このとき ip_forward は初期値の 0）
   ↓
systemd が、systemd-sysctl サービスを動かす
   ↓
systemd-sysctl が、設定ファイルを決まった順番で読む
   /etc/sysctl.d/99-router.conf に「net.ipv4.ip_forward = 1」
   ↓
書かれた値をカーネルに設定する（sysctl -w を自動でやってくれるのと同じ）
   ↓
ip_forward が 1 になる
```

[persistent-config.md](../concepts/persistent-config.md) に書いた「**永続化の正体は、起動時に毎回同じコマンドを自動で実行してもらうこと**」の、カーネルの設定版である。labs/01 で IP アドレスを永続化したときと、考え方は同じである。

| | IP アドレス（labs/01） | カーネルの設定（labs/02） |
|---|---|---|
| 一時的に変える | `sudo ip addr add ...` | `sudo sysctl -w ...` |
| 永続化するファイル | `/etc/network/interfaces` | `/etc/sysctl.d/*.conf` |
| 起動時に読むもの | ifupdown（`networking` サービス） | `systemd-sysctl` サービス |

起動時に読んだサービスの状態は、次のコマンドで確認できる。

```bash
systemctl status systemd-sysctl
```

### ファイルを作っただけでは、今のカーネルには反映されない

設定ファイルは「**次に起動したとき**」に読まれる。今すぐ反映したいときは、次のどちらかで読み直す。

```bash
sudo sysctl --system                         # 設定ファイルをすべて、決まった順番で読み直す
sudo sysctl -p /etc/sysctl.d/99-router.conf  # このファイルだけを読む
```

`sudo sysctl --system` は、どのファイルをどの順番で読んだかを表示してくれる（`* Applying /etc/sysctl.d/99-router.conf ...`）。

### 最後に必ず再起動して確かめる

`sysctl --system` で反映されても、**起動時にも正しく読まれるか**は、再起動しないと分からない。labs/01 と同じく、最後に再起動して確かめる。

```bash
sudo reboot
sudo sysctl net.ipv4.ip_forward    # 再起動後も = 1 になっているか
```

> 💡 **`/etc/sysctl.conf` について**：昔は `/etc/sysctl.conf` という 1 つのファイルに書くのが一般的だった。Debian 13 では、このファイルは標準では存在しない。古い記事に `/etc/sysctl.conf` と書かれていても、`/etc/sysctl.d/` に `.conf` のファイルを作れば同じことができる。

---

## 質問51：ファイル名と書式の決まり

### ファイル名

| 決まり | 内容 | 例 |
|---|---|---|
| **拡張子は `.conf`（必須）** | `.conf` で終わらないファイルは読まれない | `99-router.conf` は読まれる。`99-router` や `99-router.txt`、`99-router.conf.bak` は無視される |
| **名前の順に読まれる** | 文字の順（`10-...` → `50-...` → `99-...`）に読む | |
| **同じ項目は、後から読んだ値が勝つ** | 2 つのファイルに同じ項目があれば、後に読まれたファイルの値になる | |

### 先頭の数字は慣習

`99-router.conf` の `99-` は、決まりではない。**読む順番を自分で決めるための慣習**である。

| 先頭の数字 | よくある使い方 |
|---|---|
| `10-` 〜 `40-` | 早めに読ませたい設定 |
| `50-` | OS やパッケージの標準の設定によく使われる |
| `60-` 〜 `90-` | パッケージや管理者の追加の設定 |
| **`99-`** | **最後のほうに読ませたい設定**。管理者が、ほかの設定より優先させたいときに使う |

`99-router.conf` は、「**最後のほうに読まれる、ルーター用の設定**」という意味になる。数字の後ろの名前（`router`）は自由だが、何の設定か分かる名前にしておくと、あとで探しやすい。

> 💡 名前の順は「文字の順」なので、`9-foo.conf` は `10-foo.conf` より**後**に読まれる（`9` は `1` より後の文字だから）。そのため、数字は `09-`・`10-` のように桁をそろえるのが一般的である。

### 書式

```ini
# Router VM：IPv4 の転送を有効にする（labs/02）
net.ipv4.ip_forward = 1
```

| 決まり | 内容 |
|---|---|
| **1 行に 1 つ** | `項目名 = 値` |
| `=` の前後の空白 | あってもなくてもよい（`net.ipv4.ip_forward=1` でも同じ） |
| 項目名の区切り | `.` が普通。`/` でもよい（`net/ipv4/ip_forward = 1`） |
| **コメント** | `#` または `;` で始まる行 |
| 空行 | 無視される |
| 先頭の `-` | `-net.foo.bar = 1` のように書くと、その項目が存在しなくてもエラーにしない |

**項目名は、`sysctl` で表示される名前そのもの**である。名前の `.` を `/` に置き換えると、`/proc/sys/` の下のファイルの場所になる（labs/02 の Result 1）。

```text
net.ipv4.ip_forward  ←→  /proc/sys/net/ipv4/ip_forward
```

正しい名前が分からないときは、一覧から探す。

```bash
sudo sysctl -a | grep forward
```

### コメントに「なぜ」を書く

設定ファイルは、あとから自分や他人が読む。値だけでなく、**なぜその値にしたのか**をコメントに書いておくと、あとで「これは消してよいのか」を判断できる。

```ini
# Router VM：Client VM（10.0.0.0/24）のパケットを WAN 側へ転送するため（labs/02）
net.ipv4.ip_forward = 1
```

---

## 質問53：読まれる場所と、優先順位

### 設定ファイルが読まれる 4 つの場所

| 優先度 | 場所 | 誰が置くか |
|---|---|---|
| 高 | `/etc/sysctl.d/` | **管理者（自分）** |
| ↑ | `/run/sysctl.d/` | 実行中に、一時的に作られるもの |
| ↓ | `/usr/local/lib/sysctl.d/` | 自分でインストールした（パッケージ以外の）ソフトウェア |
| 低 | `/usr/lib/sysctl.d/` | **Debian のパッケージ**（OS の標準の設定） |

### 読み方は 2 段階

「優先度の高い場所が勝つ」のは、**同じファイル名のとき**だけである。実際の読み方は、次の 2 段階になっている。

```text
【① ファイル名ごとに、どの場所のファイルを使うか決める】

   同じファイル名が複数の場所にあれば、優先度の高い場所のものだけを使う
   優先度：/etc ＞ /run ＞ /usr/local/lib ＞ /usr/lib

          ↓

【② 残ったファイルを、場所に関係なく「ファイル名の順」に読む】

   同じ項目が複数のファイルにあれば、後から読んだ値が勝つ
```

### 例 1：同じファイル名なら、`/etc` が勝つ

```text
/usr/lib/sysctl.d/50-default.conf    ← 使われない（丸ごと無視される）
/etc/sysctl.d/50-default.conf        ← こちらだけが使われる
```

これが、「`/etc` が最優先」と言える場面である。パッケージのファイルを、**丸ごと自分のファイルに置き換える**ことになる。

### 例 2：違うファイル名なら、ファイル名の順で後のものが勝つ

```text
読む順番    ファイル                             中身
   1        /etc/sysctl.d/10-mine.conf           net.ipv4.ip_forward = 1
   2        /usr/lib/sysctl.d/50-pkg.conf        net.ipv4.ip_forward = 0
                                                 ─────────────────────────
                                                 結果：0（パッケージの値が勝つ）
```

`10-mine.conf` は `/etc` にあるが、ファイル名の順番が先なので、先に読まれる。後から読まれた `/usr/lib` の `50-pkg.conf` の値で上書きされる。

**「`/etc` が最後に読まれる」わけではない。** 読む順番を決めるのは、場所ではなく**ファイル名**である。

### 例 3：`99-` を付けると、自分の設定が勝つ

```text
読む順番    ファイル                             中身
   1        /usr/lib/sysctl.d/50-pkg.conf        net.ipv4.ip_forward = 0
   2        /etc/sysctl.d/99-router.conf         net.ipv4.ip_forward = 1
                                                 ─────────────────────────
                                                 結果：1（自分の値が勝つ）
```

自分のファイルの名前を `99-` で始めれば、ほかのほとんどのファイルより後に読まれる。そのため、自分の設定が勝つ。

### 覚え方

| 場面 | 勝つもの | 決め手 |
|---|---|---|
| **同じファイル名** | `/etc` のファイル | **場所** |
| **違うファイル名で、同じ項目** | 後から読まれたファイル | **ファイル名の順** |

自分の設定は、**`/etc/sysctl.d/` に、`99-` で始まる名前で書く**。こうしておけば、どちらの場面でも自分の設定が勝つ。

### 実際の読まれ方を確かめる

```bash
systemd-analyze cat-config sysctl.d
```

読まれるファイルを、**読む順番どおりに、中身ごと**表示してくれる。自分のファイルが本当に読まれているか、ほかのファイルに上書きされていないかを確かめられる。

`sudo sysctl --system` の出力（`* Applying ...` の行）でも、読んだ順番が分かる。

---

## 質問54：`/etc` 以外には、自分の設定を書かない

### 自分の設定は `/etc` に書く

| 場所 | 目的 | 自分の設定を書くと |
|---|---|---|
| **`/etc/sysctl.d/`** | **管理者（自分）の設定** | **ここに書く** |
| `/run/sysctl.d/` | 実行中だけの一時的な設定 | **再起動で消える**。`/run` はメモリの上にあるので、永続化にならない |
| `/usr/local/lib/sysctl.d/` | 自分でインストールしたソフトウェアが置く設定 | 動くが、目的が違うので、あとで探しにくくなる |
| `/usr/lib/sysctl.d/` | Debian のパッケージが置く標準の設定 | パッケージの更新で**上書きされたり、消えたりする**ことがある |

### パッケージの設定を変えたいとき

`/usr/lib/sysctl.d/` の設定を変えたい場合も、そのファイルは**直接編集しない**。`/etc` の側で、次のどちらかを行う。

| やり方 | 方法 | 結果 |
|---|---|---|
| **一部の項目だけ変える** | 同じ項目を、`/etc/sysctl.d/99-...conf` に書く | パッケージの設定は残り、その項目だけ自分の値が後から勝つ |
| **ファイルを丸ごと置き換える** | 同じファイル名のファイルを、`/etc/sysctl.d/` に作る | パッケージのファイルは丸ごと使われなくなる |

普段は「一部の項目だけ変える」ほうを使う。丸ごと置き換えると、パッケージが将来その設定に追加した項目も、使われなくなってしまうからである。

### sysctl だけの決まりではない

「**パッケージの設定は `/usr`、管理者の設定は `/etc`**」という分け方は、Linux 全体の考え方である。

```text
/usr  … OS やパッケージのもの。パッケージ管理（apt）が管理する。自分では触らない
/etc  … このマシンだけの設定。管理者が書く。パッケージの更新でも消されない
/run  … 動いている間だけのもの。再起動で消える
```

| 設定 | パッケージの標準 | 管理者の設定 |
|---|---|---|
| カーネルの設定 | `/usr/lib/sysctl.d/` | `/etc/sysctl.d/` |
| systemd のサービス | `/usr/lib/systemd/system/` | `/etc/systemd/system/` |
| ネットワーク（ifupdown） | — | `/etc/network/interfaces` |
| ファイアウォール（nftables） | — | `/etc/nftables.conf` |

「自分の設定は `/etc`、`/usr` は触らない」と覚えておけば、ほかの設定でも迷いにくい。

---

## 質問52：永続化の方法が書かれているサイト

### 全部がまとまったサイトは無い

永続化の方法は、設定ごとに、**起動時にその設定を読むソフトウェア**の公式ドキュメントに書かれている。調べ方のコツは、次の順番で考えることである。

```text
① 変えたい設定は何か              例：IP アドレス
        ↓
② 起動時に、それを設定するのは誰か  例：ifupdown（networking サービス）
        ↓
③ その説明書（man ページ）を読む    例：man interfaces
```

### このラボで使う設定と、その説明書

| 設定 | 永続化するファイル | 起動時に読むもの | 説明書 |
|---|---|---|---|
| IP アドレス（labs/01） | `/etc/network/interfaces` | ifupdown（`networking` サービス） | `man interfaces` |
| カーネルの設定（labs/02） | `/etc/sysctl.d/*.conf` | `systemd-sysctl` | `man sysctl.d` |
| ファイアウォール・NAT（labs/04・05） | `/etc/nftables.conf` | `nftables` サービス | `man nft` |
| サービスの自動起動 | `systemctl enable` で設定 | systemd | `man systemctl` |

### man ページの読み方

Router VM の中で、次のように打てば読める。

```bash
man sysctl.d
man interfaces
```

man ページの最後のほうにある **「FILES」の節**に、「どのファイルを読むか」が書かれている。初めて使うソフトウェアでも、`man <名前>` の FILES を見れば、永続化の場所が分かることが多い。

ブラウザで読むなら、**Debian の man ページのサイト**（<https://manpages.debian.org/>）がある。Debian のバージョンに合ったものを選べる。

### まとまった解説を読むなら

| サイト | 内容 |
|---|---|
| [Debian リファレンス](https://www.debian.org/doc/manuals/debian-reference/) | Debian 公式の解説書。システムの初期化（第 3 章）やネットワークの設定（第 5 章）など。日本語版もある |
| [Debian Wiki：NetworkConfiguration](https://wiki.debian.org/NetworkConfiguration) | `/etc/network/interfaces` の書き方の例が多い |
| [Linux カーネル：ip-sysctl](https://docs.kernel.org/networking/ip-sysctl.html) | `net.ipv4.ip_forward` など、ネットワーク関係のカーネルの設定項目の意味 |

### ネットの記事を読むときの注意

ネットの記事は、次のような理由で、手元の環境と合わないことがある。

- 古い書き方（`/etc/sysctl.conf` など）
- Debian 以外のディストリビューション（Ubuntu の netplan、RHEL の NetworkManager など）の方法

記事を参考にするときも、**最後は手元の `man` で確かめる**のが確実である。

---

## まだ試していないこと

| コマンド（Router VM） | 予想される結果 | 確かめられること |
|---|---|---|
| `ls /usr/lib/sysctl.d/ /etc/sysctl.d/` | `/usr/lib/sysctl.d/` にパッケージのファイル（`50-...conf` など）が、`/etc/sysctl.d/` には自分で作ったもの（または `README` だけ）がある | パッケージの設定と、管理者の設定の置き場所が分かれている（質問53・54） |
| `systemd-analyze cat-config sysctl.d` | 読まれるファイルが、ファイル名の順に中身ごと表示される | 読む順番は、場所ではなくファイル名で決まる（質問53） |
| `sudo sysctl --system` | `* Applying /usr/lib/sysctl.d/...` や `* Applying /etc/sysctl.d/99-router.conf ...` が、ファイル名の順に並ぶ | 同上 |
| `systemctl status systemd-sysctl` | `active (exited)` になっている | 起動時に一度だけ設定を読んで、仕事を終えたサービスである（質問50） |
| `/etc/sysctl.d/99-router.conf` を作って再起動 → `sudo sysctl net.ipv4.ip_forward` | `net.ipv4.ip_forward = 1` | 永続化できている（質問50） |
| 試しに `/etc/sysctl.d/99-test` （`.conf` を付けない）に設定を書いて `sudo sysctl --system` | `99-test` は `Applying` の一覧に出てこない | ファイル名は `.conf` で終わる必要がある（質問51）。試したら消しておく |

---

## まとめ

- **カーネルの設定は、`/etc/sysctl.d/` に `.conf` のファイルを作れば永続化できる。** 起動時に `systemd-sysctl` が読んで、カーネルに設定する。今すぐ反映するには `sudo sysctl --system`、最後に必ず再起動して確かめる。
- ファイル名は **`.conf` で終わる**必要があり、**名前の順に読まれる**。同じ項目は、後から読んだ値が勝つ。先頭の数字（`99-` など）は、読む順番を決めるための慣習である。
- 書式は **1 行に 1 つ `項目名 = 値`**。`#` か `;` で始まる行はコメント。項目名は `sysctl` で表示される名前そのもの。
- 読まれる場所は 4 つ（`/etc`、`/run`、`/usr/local/lib`、`/usr/lib`）。**同じファイル名なら場所で決まり（`/etc` が勝つ）、違うファイル名ならファイル名の順で決まる（後が勝つ）。** 「`/etc` が最後に読まれる」わけではない。
- だから、自分の設定は **`/etc/sysctl.d/` に、`99-` で始まる名前で書く**。
- `/run` は再起動で消え、`/usr/lib` はパッケージの更新で上書きされることがある。**パッケージの設定は `/usr`、管理者の設定は `/etc`** は、Linux 全体の考え方である。
- 永続化の方法がまとまったサイトは無い。「**起動時にその設定を読むのは誰か**」を考え、その man ページ（特に FILES の節）を読む。

---

## 参考

- [sysctl.d(5)：設定ファイルの場所、ファイル名、書式、読む順番](https://manpages.debian.org/trixie/systemd/sysctl.d.5.en.html)
- [systemd-sysctl.service(8)](https://manpages.debian.org/trixie/systemd/systemd-sysctl.service.8.en.html)
- [sysctl(8)：`-w`、`-p`、`--system`](https://manpages.debian.org/trixie/procps/sysctl.8.en.html)
- [systemd-analyze(1)：`cat-config`](https://manpages.debian.org/trixie/systemd/systemd-analyze.1.en.html)
- [interfaces(5)：`/etc/network/interfaces` の書式](https://manpages.debian.org/trixie/ifupdown/interfaces.5.en.html)
- [Filesystem Hierarchy Standard（`/etc`、`/usr`、`/run` の役割）](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html)
- [Debian リファレンス](https://www.debian.org/doc/manuals/debian-reference/)
- [Linux カーネル：ip-sysctl](https://docs.kernel.org/networking/ip-sysctl.html)
