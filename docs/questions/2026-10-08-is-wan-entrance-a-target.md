# WANの入口は攻撃者の標的になるのか

調査日：2026-10-08

## 疑問

> 基本的なネットワークの考え方として、WANの入口は攻撃者の標的となり得るか？

## 結論

**なる。WANの入口は、ネットワークの中で最も狙われやすい場所の1つである。**

| ポイント | 一言でいうと |
|---|---|
| 誰でも届く | インターネット上の誰でも、WAN側のIPアドレスへパケットを送れる |
| すぐ見つかる | 世界中のスキャナーが、IPアドレスを片っ端から調べ続けている |
| 突破されると被害が大きい | ルーターはLAN全体の出入口なので、乗っ取られるとLAN全体が危険になる |

ただし、**「WANだけ守れば安全」ではない**。攻撃はLAN側（内側）から始まることもある（下の「WANだけ守ればよいわけではない」を参照）。

---

## 前提：WANの入口とは

```text
           Internet（誰でもいる）
                  │
                  │   ← ここが「WANの入口」
                  ▼
        ┌──────────────────┐
        │ WAN NIC          │   ← 外から見える面
        │                  │
        │      Router      │
        │                  │
        │ LAN NIC          │
        └────────┬─────────┘
                 │
       ┌─────────┼─────────┐
       PC      Server     IoT     ← 守りたいもの（LAN）
```

家にたとえると次のようになる。

| ネットワーク | 家にたとえると |
|---|---|
| インターネット | 家の前の道路（誰でも通れる） |
| WANの入口 | 玄関 |
| WAN側で開いているポート | 開いている窓やドア |
| ルーター / Firewall | 玄関の鍵・インターホン |
| LAN | 家の中 |
| スキャン | 一軒ずつドアノブを回して回る人 |

玄関は道路に面しているので、誰でもノックできる。鍵が弱かったり窓が開いていたりすれば、入られてしまう。

### 用語

| 用語 | 意味 |
|---|---|
| ポート | 1台の機器の中で、通信の受付窓口を区別する番号（SSHなら22番など） |
| スキャン | 大量のIPアドレス・ポートへ通信を送り、反応がある機器を探すこと |
| 攻撃対象領域（Attack Surface） | 外から触れられる場所の総量。開いているポートやサービスが多いほど広くなる |
| ハニーポット | 攻撃を観察するために、わざと公開しておく「おとり」のサーバー |
| ブルートフォース（総当たり） | パスワードを大量に試してログインを狙う攻撃 |
| DDoS | 大量の機器から一斉に通信を送りつけ、サービスを止める攻撃 |

---

## なぜ狙われるのか

### 1. 誰でも届く

LAN内の機器はルーターの内側にいるが、WAN側のIPアドレスはインターネットに直接さらされている。攻撃者は世界のどこからでもパケットを送れる。

### 2. 自動で、すぐに見つかる

「自分のIPアドレスなんて誰も知らないから大丈夫」は成り立たない。

| 事実 | 内容 |
|---|---|
| IPv4全体のスキャン | 研究用スキャナーZMapは、1台のマシンからIPv4アドレス空間全体を**45分以内**（1GbE）、10GbEなら**5分以内**でスキャンできる |
| 常にスキャンしているサービス | Shodan、Censysなどが、インターネット上の機器を調べ続けて一覧化している |

攻撃者は特定の誰かを狙っているのではなく、**弱いところがある機器を機械的に探している**。WANの入口を持っている時点で、その探索の対象に入る。

### 3. 突破したときの見返りが大きい

ルーターはLANのすべての通信が通る場所である。乗っ取られると、

- LAN内の機器へ攻撃を広げる足場にされる
- 通信を盗み見・改ざんされる
- ボットネットの一員としてDDoS攻撃に使われる

といった被害につながる。

実際に2025年2月、米国のCISAやオーストラリアのASDなど各国のセキュリティ機関が、Firewall・ルーター・VPN Gatewayなどの**エッジデバイス（ネットワークの境界にある機器）**が攻撃者に狙われているとして、共同でガイダンスを公開している。

---

## 実際どれくらい狙われるか

### 事例1：インターネットに公開したおとりサーバー（Unit 42、2021年）

Palo Alto NetworksのUnit 42が、ハニーポットを世界中のクラウドに公開して観察した。

| 項目 | 結果 |
|---|---|
| 期間 | 2021年7〜8月 |
| ハニーポット | 320台（SSH / RDP / SMB / Postgres） |
| 設定 | 一部のアカウントに、わざと弱いパスワード（`admin:admin` など）を設定 |
| 24時間以内に侵害された割合 | **80%** |
| 1週間以内 | **全台**が侵害された |
| 最も狙われたサービス | **SSH**（1台あたり平均で1日26回侵害された） |
| SSHで最初に侵害されるまで | 平均 約3時間 |
| 最速の例 | 1つの攻撃者が、Postgresのハニーポット80台の96%を**30秒以内**に侵害した |

> わざと弱い設定にした実験なので、普通の機器がすぐに乗っ取られるわけではない。
> ただし、**公開した瞬間から攻撃が届き始める**ことは、設定の強さに関係なく同じである。

### 事例2：Mirai（2016年〜）

ルーターやネットワークカメラなどのIoT機器を乗っ取ったマルウェア。

```text
① ランダムなIPv4アドレスのTelnet（23 / 2323番）へ接続を試す
        ↓
② 反応があれば、工場出荷時のID / パスワード（約60組）でログインを試す
   例：admin / admin、root / 12345
        ↓
③ ログインできたら乗っ取り、その機器にも同じ探索をさせて感染を広げる
        ↓
④ 乗っ取った大量の機器から一斉にDDoS攻撃
```

**WAN側に管理用のポートが開いていて、パスワードが初期値のまま**という機器がまとめて狙われた。

---

## WANの入口の「何が」狙われるか

| 狙われるもの | 例 | 主な対策 |
|---|---|---|
| WAN側に開いている管理用ポート | SSH（22）、Telnet（23）、Web管理画面 | 管理用の入口は、インターネットから直接触れないようにする |
| 初期パスワード・弱いパスワード | `admin / admin`（Mirai） | 初期パスワードを変える。公開鍵認証やMFAを使う |
| 修正されていない脆弱性 | VPN機器やFirewallの脆弱性 | 更新を適用する。サポートが切れた機器は使い続けない |
| 不要なサービス | 使っていないVPN、UPnPなど | 使わない機能・ポートは無効にする |
| 回線そのもの | DDoS | 自分の機器だけでは防げないので、ISPなど上流での対策も必要 |

対策の欄は、CISA / ASDなどのガイダンスで推奨されている内容に沿っている。

一番のポイントは、**WAN側で開けるものを最小限にする**こと。開いていないポートは、スキャンされても入口にならない。

---

## WANだけ守ればよいわけではない

攻撃は外から来るものだけではない。

```text
            Internet
             │    ▲
          ①  │    │  ③
             ▼    │
          ┌──────────┐
          │  Router  │
          └──────────┘
             │    ▲
             │    │  ②
             ▼    │
              LAN
```

| 方向 | 例 | 主な対策 |
|---|---|---|
| ① 外 → 内 | スキャン、パスワード総当たり、脆弱性攻撃 | WAN側から始まる通信は原則として拒否する |
| ② 内 → ルーター / 内 → 内 | 乗っ取られたPCやIoT機器が、ルーターの管理画面や他の機器を狙う | 管理アクセスできる機器を限定する。強い認証を使う |
| ③ 内 → 外 | 不審なリンクから感染したマルウェアが、攻撃者のサーバーと通信する | 必要な通信だけ許可する。ログを確認する |

このように、入口を1つ固めるだけでなく、複数の層で守る考え方を**多層防御（Defense in Depth）**という。

---

## よくある誤解：「NATがあるから安全」

家庭用ルーターでは、外から始まる通信がLAN内の機器に届かないことが多い。これを「NATが守ってくれている」と考えがちだが、正確ではない。

IPv6のネットワーク保護について書かれたRFC 4864では、次のように説明されている。

- アドレスを書き換えること自体は、セキュリティを提供しない
- 守られているように見えるのは、外から来た通信に対応する変換の記録（状態）が無いため
- 同じ保護は、基本的なFirewallでも得られる

| 仕組み | 本来の役割 | 外からの通信を防いでいるか |
|---|---|---|
| NAT | 複数の機器で1つのIPアドレスを共有する | 結果的に防いでいるように見えるだけ |
| Stateful Firewall | 通信の状態を見て、許可・拒否する | **これが本来の防御** |

特にIPv6ではNATを使わないことが多いため、**Firewallで外からの通信を拒否していないと、LAN内の機器がインターネットから直接見えてしまう**。

---

## このラボではどうなっているか

現在のラボで「WAN」と呼んでいるのは、VMwareのNATネットワーク（VMnet8）である。

```text
Internet
   │
Windows（VMware NAT）    ← 外から始まる通信はここで止まる
   │
VMnet8
   │
debian-router WAN NIC    ← ラボ上の「WAN」
```

| 項目 | 今のラボ | 将来の実機 |
|---|---|---|
| WANの先 | VMware NAT（VMnet8） | ISP経由でインターネット |
| インターネットから直接届くか | 届かない（VMware側でポートフォワードを設定していない場合） | **届く** |
| WAN側のSSH | 管理用に使っている（[remote-access](../../environment/remote-access/README.md)） | 開けたままにすると、すぐにスキャン・総当たりの対象になる |

そのため、今の構成でWAN側からSSHしていても、インターネット上の攻撃者からは見えない。

一方で、実機をインターネットにつなぐときは、**WAN側の22番ポートを開けたままにしない**ように、管理用の経路を見直す必要がある（例：LAN側から管理する、VPN経由で管理する）。

関連するLab：

- [05 - Firewall](../../labs/05-firewall/README.md)：WAN側から始まる通信を拒否するルールを作る
- [08 - IPv6](../../labs/08-ipv6/README.md)：NATが無い環境でのFirewall
- [10 - VPN](../../labs/10-vpn/README.md)：WANに管理用ポートを開けずに、外から管理する方法

---

## まとめ

- **WANの入口は狙われる。** インターネットから誰でも届き、自動スキャンで短時間のうちに見つかる。
- 狙われるのは主に**開いているポート・弱いパスワード・古いソフトウェア**。WAN側で開けるものを最小限にする。
- **NATはFirewallではない。** 外からの通信を防いでいるのは、通信の状態を見て判断するFirewallの働き。
- **内側からの攻撃もある。** WANだけでなく、LAN側や外向きの通信も含めて多層で守る。

---

## 参考資料

- Durumeric, Wustrow, Halderman. [ZMap: Fast Internet-Wide Scanning and its Security Applications](https://zmap.io/paper.pdf)（USENIX Security 2013）
- [ZMap README](https://github.com/zmap/zmap)
- Unit 42. [Observing Attacks Against Hundreds of Exposed Services in Public Clouds](https://unit42.paloaltonetworks.com/exposed-services-public-clouds/)（2021-11-22）
- BleepingComputer. [Threat actors find and compromise exposed services in 24 hours](https://www.bleepingcomputer.com/news/security/threat-actors-find-and-compromise-exposed-services-in-24-hours/)
- CSO Online. [Here are the 61 passwords that powered the Mirai IoT botnet](https://www.csoonline.com/article/558215/here-are-the-61-passwords-that-powered-the-mirai-iot-botnet.html)
- [The Impact of DoS Attacks on Resource-constrained IoT Devices: A Study on the Mirai Attack](https://arxiv.org/pdf/2104.09041)
- CISA. [CISA, Partners Release Guidance on Edge Devices](https://www.cisa.gov/news-events/alerts/2025/02/04/cisa-partners-asds-acsc-cccs-ncsc-uk-and-other-international-and-us-organizations-release-guidance)（2025-02-04）
- ASD's ACSC. [Mitigation strategies for edge devices: executive guidance](https://www.cyber.gov.au/business-government/protecting-devices-systems/hardening-systems-applications/network-hardening/securing-edge-devices/mitigation-strategies-for-edge-devices-executive-guidance)
- [RFC 4864: Local Network Protection for IPv6](https://www.rfc-editor.org/rfc/rfc4864)
