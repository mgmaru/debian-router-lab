# IPアドレスを知られると侵入されるのか（Tailscale・VMのIP）

調査日：2026-10-08

関連：

- [WANの入口は攻撃者の標的になるのか](2026-10-08-is-wan-entrance-a-target.md)
- [プライベートIPとグローバルIP：知られると危ないのはどちらか](2026-10-08-private-ip-vs-global-ip.md)

## 疑問

このリポジトリは公開されており、ドキュメントにTailscaleのIPやVMのIPが書かれている。そこで「IPを隠すべきか」という議論になった。

> **質問2**：TailscaleのIPが分かったとしても、Tailscaleはアカウント認証があるので、同じネットワークには外部から入られないのでは？

> **質問3**：VMのWANおよびLANのIPが知られた場合、VMのWANから侵入されてしまうのか？
> 有り得るが可能性は少ないと思う。そもそも、同じネットワーク上（Tailscale上のネットワーク）にいる必要があるのでは？

## 結論

| 質問 | 答え |
|---|---|
| 質問2 | **ほぼその通り。** Tailscale IPはインターネットから届かず、tailnet（Tailscaleのネットワーク）に参加した機器からしか通信できない。ただし、**アカウントそのものが乗っ取られた場合**は別の話になる |
| 質問3 | **考え方は正しい。** VMのIPはプライベートIPなので、インターネットからは届かない。補足すると、Tailscaleに入っただけでもVMのIPには直接届かない。届くのは**Windows本体の中**からだけである |

---

## 前提：IPアドレスは「住所」であって「鍵」ではない

侵入するには、次の3つがすべてそろう必要がある。

| 条件 | 意味 | 家にたとえると |
|---|---|---|
| ① 宛先が分かる | IPアドレスを知っている | 住所を知っている |
| ② 届く | そのIPアドレスまでパケットが届く経路がある | その家まで行ける道がある |
| ③ 入れる | パスワードや鍵などの認証を突破できる | 玄関の鍵を開けられる |

**IPアドレスを知られるのは①だけ**である。②と③がそろわなければ侵入できない。

```text
IPを知っている（①）
      │
      ▼
 そこまで届く？（②）── No ──▶ 侵入できない
      │ Yes
      ▼
 認証を突破できる？（③）── No ──▶ 侵入できない
      │ Yes
      ▼
   侵入される
```

しかも①は、インターネットに面しているIPであればスキャンで簡単に見つかってしまう（[WANの入口は攻撃者の標的になるのか](2026-10-08-is-wan-entrance-a-target.md) を参照）。
そのため、守りの中心はIPを隠すことではなく、**②届かないようにすること**と**③入れないようにすること**になる。

---

## 質問2：TailscaleのIPを知られたら？

### Tailscale IPはインターネットから届かない

Tailscaleの機器には `100.x.y.z` 形式のIPアドレスが割り当てられる。

| 項目 | 内容 |
|---|---|
| 範囲 | `100.64.0.0/10`（RFC 6598で定められたCGNAT用の範囲） |
| インターネットでの扱い | インターネットのルーターは、この範囲を転送しない |
| Tailscaleの説明 | Tailscale IPはパブリックなインターネットには公開されない |

```text
インターネット上の攻撃者 ──✕──▶ 100.80.124.76   （届かない）

tailnetに参加した機器   ──✓──▶ 100.80.124.76   （届く）
```

### tailnetに入るにはアカウント認証が必要

```text
新しい機器
   │  Tailscaleアプリでログイン
   ▼
IDプロバイダー（Google / GitHub / Microsoftなど）で認証
   │
   ▼
（Device approvalを有効にしていれば）管理者が承認
   │
   ▼
tailnetに参加 → 他の機器の 100.x.y.z へ通信できる
```

つまり質問の通り、**IPを知っているだけではtailnetに入れない**。

### 注意点：本当の入口は「IP」ではなく「アカウント」

Tailscaleで守るべきものは、IPアドレスではなく**アカウントと、tailnetに参加している機器**である。

| リスク | 何が起きるか | 対策 |
|---|---|---|
| IDプロバイダーのアカウントが乗っ取られる（フィッシングなど） | 攻撃者が自分の機器をtailnetに追加できる。初期設定のポリシーではtailnet内の全機器が互いに通信できるため、WindowsのSSHまで届く | IDプロバイダーでMFAを有効にする（できればハードウェアキー）。Device approvalで新しい機器を承認制にする |
| tailnet内の機器が乗っ取られる | 例：MacBookがマルウェアに感染すると、そこからWindowsへ届く | ACL / grantsで必要な通信だけを許可する。各機器を最新に保つ |
| 機器の鍵が盗まれる | 盗まれた鍵で機器になりすまされる | Key expiry（鍵の有効期限）で定期的に鍵を更新させる |
| Funnelを有効にする | 機器のサービスがインターネットに公開され、URLを知っていれば誰でもアクセスできる | 意図せず有効にしない |

### 仮にtailnetに入られても、まだ壁がある

tailnetに入れても、Windowsに入るにはさらに**WindowsへのSSHログイン**が必要である。

ただし、このリポジトリにはWindowsのユーザー名（`kajih`）が書かれている。**パスワード認証のままだと、残る壁はパスワードだけ**になる。
[remote-access](../../environment/remote-access/README.md) に「将来的な公開鍵認証」として書いている通り、公開鍵認証に切り替えると、ユーザー名が知られていてもほぼ影響がなくなる。

---

## 質問3：VMのWAN / LANのIPを知られたら？

### VMのIPはプライベートIP

| VMのIP | 範囲 | インターネットから |
|---|---|---|
| WAN：`192.168.x.x`（VMnet8） | RFC 1918のプライベートIP | 届かない |
| LAN：`10.0.0.0/24`（VMnet2） | RFC 1918のプライベートIP | 届かない |

RFC 1918では、プライベートIPのパケットはインターネットを越えて転送しないこと、ISPなどのルーターはこれを拒否するように設定することが定められている。

また、`192.168.x.x` や `10.0.0.x` は、世界中の家庭や会社で同じように使われている。知られても、**どこにある機器なのかすら特定できない**。

### 誰ならVMに届くのか

```text
[Internet] ──✕── VMには届かない（プライベートIP / VMware NAT）

[tailnet] ──Tailscale──▶ [Windows 11]   ← tailnetから届くのはここまで
                              │
                              │ VMnet8用の仮想アダプター
                              ▼
                          [VMnet8]
                              │
                       [debian-router]
                              │
                          [VMnet2]
                              │
                       [debian-client]
```

| どこから | VMのWAN（VMnet8）に届くか | 理由 |
|---|---|---|
| インターネット | ✕ | プライベートIPは転送されない。VMware NATも、ポートフォワードを設定していなければ外から始まる通信を転送しない |
| Windowsと同じLANにいる別の機器 | ✕（通常） | VMnet8はWindowsの中にあるNATネットワークのため |
| tailnet上の機器（MacBookなど） | ✕（直接は届かない） | WindowsをSubnet routerにしてVMnet8を公開していないため。届くのはWindowsまで |
| Windows本体 | ✓ | VMnet8用の仮想アダプターを持っている |

質問の「同じネットワーク上にいる必要がある」は正しい。より正確には、**VMのIPに届く経路を持つ場所＝Windows本体の中**にいる必要がある。Tailscaleは、Windowsまでの通り道にすぎない。

### 攻撃者がVMに入るまでの道のり

```text
攻撃者
  │ ① tailnetに入る
  │    （IDプロバイダーのアカウント + MFA + Device approval を突破）
  ▼
tailnet
  │ ② WindowsにSSHでログイン
  │    （Windowsのパスワード / 鍵を突破）
  ▼
Windows 11
  │ ③ debian-routerにSSHでログイン
  │    （VMのパスワード / 鍵を突破）
  ▼
debian-router
  │ ④ debian-clientにSSHでログイン
  ▼
debian-client
```

IPアドレスを知っていても、**どの段階でも「届く」と「認証」の両方が必要**になる。

### 「有り得るが可能性は少ない」について

この見立ても正しい。ただし、現実的な侵入経路はIPアドレスではなく、次のようなものになる。

| 経路 | 例 |
|---|---|
| アカウントの乗っ取り | Googleなどのアカウントがフィッシングで盗まれ、tailnetに機器を追加される |
| 機器の乗っ取り | WindowsやMacBookがメールやWebサイト経由でマルウェアに感染する |

特に**Windowsが感染した場合、Tailscaleの認証を経由せずに上の図の②の位置から始められる**。これは、前回の疑問でいう「内側からの攻撃」にあたる。

### 実機になったら事情が変わる

| 項目 | 今のラボ | 将来の実機 |
|---|---|---|
| WANのIP | プライベートIP（VMnet8） | ISPから割り当てられるIP（多くはグローバルIP） |
| インターネットから届くか | 届かない | **届く** |
| IPを知られる影響 | ほぼない | 隠してもスキャンで見つかるので、隠すこと自体に意味がない |

実機では「届かない」という壁がなくなるため、**WAN側で開けるポートを最小限にする**ことと、**認証を強くする**ことがより重要になる。

---

## 公開してよい情報・いけない情報

今回の「IPを隠すかどうか」の議論をまとめると、次のようになる。

| 情報 | 公開されたときの影響 | 判断 |
|---|---|---|
| Tailscale IP（`100.80.124.76`） | tailnetの外からは届かない。単独では侵入に使えない | 公開していても大きな問題はない |
| VMのIP（`192.168.x.x`、`10.0.0.x`） | プライベートIP。ありふれた値で、場所の特定もできない | 問題ない |
| ユーザー名（`kajih`）、コンピューター名 | 届いた後の総当たりで、ユーザー名を推測する手間が省ける | 単独では問題ないが、パスワード認証のままだと壁が1枚減る。公開鍵認証にすれば影響はほぼない |
| パスワード、秘密鍵、Tailscaleの認証キー | 届いた時点で入られる。tailnetに機器を追加される | **絶対に公開しない** |

ポイントは、**IPアドレスを隠すこと（Security through Obscurity）に頼らない**こと。

```text
守りの優先順位

  ③ 入れない（認証）       ← 一番大事：MFA、公開鍵認証
  ② 届かない（経路）       ← 不要なポートを開けない、ネットワークを分ける
  ① 知られない（隠す）     ← おまけ程度
```

---

## まとめ

- **IPアドレスは住所であって、鍵ではない。** 侵入には「届く」と「認証の突破」が必要。
- **質問2**：その通り。Tailscale IPはtailnetの外から届かない。守るべきはIPではなく、**IDプロバイダーのアカウント（MFA）とtailnet内の機器**。
- **質問3**：その通り。VMのIPはプライベートIPで、届くのは**Windows本体の中**からだけ。現実的なリスクはIPを知られることではなく、**アカウントや機器の乗っ取り**。
- 効果が大きい対策は、**IDプロバイダーのMFA**と**SSHの公開鍵認証**。

---

## 参考資料

- Tailscale. [What are these 100.x.y.z addresses?](https://tailscale.com/docs/concepts/tailscale-ip-addresses)
- Tailscale. [What devices can connect to or know mine?](https://tailscale.com/docs/concepts/device-visibility)
- Tailscale. [ACL policy examples](https://tailscale.com/docs/reference/examples/acls)
- Tailscale. [Best practices to secure your tailnet](https://tailscale.com/docs/reference/best-practices/security)
- Tailscale. [Subnet routers](https://tailscale.com/kb/1019/subnets)
- Tailscale. [Tailscale Funnel](https://tailscale.com/docs/features/tailscale-funnel)
- [RFC 6598: IANA-Reserved IPv4 Prefix for Shared Address Space](https://www.rfc-editor.org/rfc/rfc6598)
- [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918)
- VMware. [Configure Port Forwarding for a NAT Network](https://docs.vmware.com/en/VMware-Workstation-Pro/17/com.vmware.ws.using.doc/GUID-A1D6B380-680B-46A8-A70D-572562EB935E.html)
