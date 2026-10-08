# Network Interface

## IP アドレスと `/24`

`10.0.0.1/24` は「IP アドレス `10.0.0.1`、ネットワーク部の長さ 24 ビット」という意味です。

- IPv4 アドレスは 32 ビット（8 ビット × 4 つ）
- `/24` は先頭 24 ビット ＝ `10.0.0` の部分が **ネットワークの名前**
- 残り 8 ビット（最後の数字）が **そのネットワーク内の個々の機器の番号**

つまり `10.0.0.0`〜`10.0.0.255` が **同じネットワーク** です。
（`10.0.0.0` はネットワーク自身、`10.0.0.255` はブロードキャスト用なので、機器に付けられるのは `.1`〜`.254`）

**同じネットワーク内の相手には、ルーターを介さず直接通信できます。** これが [デフォルトゲートウェイ](routing.md#デフォルトゲートウェイ) の話の前提になります。

---

## `ip a` の読み方

```bash
ip a        # ip addr の省略形
```

ルーター VM での出力例（抜粋）：

```
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ... state UP ...
    link/ether 00:0c:29:73:2d:61 brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    altname enx000c29732d61
    inet 192.168.22.129/24 brd 192.168.22.255 scope global dynamic noprefixroute ens33
       valid_lft 1618sec preferred_lft 1393sec
3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ... state UP ...
    link/ether 00:0c:29:73:2d:6b brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/24 scope global ens37
       valid_lft forever preferred_lft forever
```

### 見るべきポイント

| 項目 | 意味 |
|---|---|
| `2: ens33` | インターフェース名。設定ファイルにはこの名前を書く |
| `state UP` | インターフェースが動いている |
| `link/ether 00:0c:29:...` | MAC アドレス。`00:0c:29` で始まるのは VMware の仮想 NIC |
| `altname enp2s1` など | 同じインターフェースの別名。設定には通常 `ens33` のほうを使う |
| `inet 192.168.22.129/24` | IPv4 アドレス |
| `dynamic` | **DHCP でもらったアドレス**。期限（`valid_lft 1618sec`）がある |
| `dynamic` が付いていない | **手で設定したアドレス**（`valid_lft forever`） |
| `noprefixroute` | DHCP クライアントなどが付けた印。気にしなくてよい |

---

## リンクローカルアドレス（`169.254.x.x`）

`169.254.0.0/16` は **リンクローカルアドレス** と呼ばれ、**DHCP で IP をもらえなかったときに自動で付けられる** アドレスです。

ネットワークに DHCP サーバーがいないと、DHCP クライアントの問い合わせに誰も答えないため、代わりにこのアドレスが付きます。

> 💡 Windows で「ネットワークにつながっているのに `169.254.x.x` になっている」ときも同じ原因（DHCP に失敗している）です。覚えておくとトラブル調査で役立ちます。

このラボで実際に観察した例は [labs/01-network-interface](../../labs/01-network-interface/README.md) を参照。
