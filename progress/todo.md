# TODO

このプロジェクトではTODOリストを用意する。

理由：

- 環境構築
- ネットワーク学習
- Router Software開発
- 実機移植

まで工程が長く、現在位置を把握しやすくするため。

## Phase 1 - Virtual Lab Setup

- [x] VMware Workstation ProをWindowsへインストール
- [x] Debian ISOをダウンロード
- [x] Debian Router VMを作成
- [x] Debian Router VMへDebianをインストール
- [x] Debian Client VMを作成
- [x] Debian Client VMへDebianをインストール
- [x] MacBookからWindowsへTailscale経由でSSH接続
- [x] WindowsからRouter VMへSSH接続
- [x] MacBookからWindows経由でRouter VMへSSH接続
- [x] LAN用VMnet2を作成
- [x] Router VMへ2枚目の仮想NICを追加
- [x] Router VMの2枚目NICをVMnet2へ接続
- [x] Client VMのNICをVMnet2へ変更
- [x] Router VM / Client VM双方でNICを確認
- [x] Router VMからClient VMへSSH接続確認
- [x] MacBookの `~/.ssh/config` にWindows / Router / Clientを登録
- [x] `ssh debian-router` でRouter VMへ接続確認
- [x] `ssh debian-client` でClient VMへ多段SSH接続確認

## Phase 2 - Basic Network Setup

- [x] Router LAN側に `10.0.0.1/24` を一時設定
- [x] Clientに `10.0.0.10/24` を一時設定
- [x] Router / ClientのIP設定を `/etc/network/interfaces` などへ記述して永続化
- [x] ClientのDefault Gatewayを `10.0.0.1` に設定
- [x] Router ↔ Client間で相互 `ping` 確認
- [ ] `ip addr` を理解
- [ ] `ip link` を理解
- [ ] `ip route` を理解
- [ ] `ip neigh` を理解
- [ ] `tcpdump` でPacketを観察

## Phase 3 - Router Fundamentals

- [ ] IP Forwardingを学習
- [ ] IP Forwardingを有効化
- [ ] Routing Tableを学習
- [ ] Client → Router → WAN のPacket経路を確認
- [ ] NATを学習
- [ ] nftablesでNATを設定
- [ ] ClientからInternetへの疎通確認
- [ ] Firewallを学習
- [ ] nftablesでFirewall Ruleを設定

## Phase 4 - Network Services

- [ ] DHCPの仕組みを学習
- [ ] DHCP Serverを構築
- [ ] ClientへIPを自動配布
- [ ] DNSの仕組みを学習
- [ ] DNS Forwarderを構築
- [ ] IPv6を学習
- [ ] VLANを学習
- [ ] VPNを学習

## Phase 5 - Automation

- [ ] 手動設定を整理
- [ ] 設定ファイルを `configs/` に保存
- [ ] Setup Scriptを作成
- [ ] Test Scriptを作成
- [ ] Reset / Cleanup Scriptを作成

## Phase 6 - Router Software

- [ ] `routerd` の責務を決定
- [ ] Interface Management
- [ ] Routing Management
- [ ] Firewall Management
- [ ] NAT Management
- [ ] Traffic Monitoring
- [ ] CLI / API
- [ ] NetlinkによるLinux Kernel制御を検討
- [ ] VM上で自動テスト

## Phase 7 - Hardware Selection

- [ ] CPU使用量測定
- [ ] Memory使用量測定
- [ ] Network負荷測定
- [ ] 必要NIC数決定
- [ ] 1GbE / 2.5GbE選定
- [ ] ARM64 / x86-64選定
- [ ] SBC / Mini PC比較
- [ ] 実機購入

## Phase 8 - Physical Router Deployment

- [ ] 実機へDebianをインストール
- [ ] `routerd` をビルド
- [ ] `configs/` を実機へ反映
- [ ] `deploy/install.sh` を作成
- [ ] systemd Service化
- [ ] 実WAN NIC接続
- [ ] 実LAN NIC接続
- [ ] Routing確認
- [ ] NAT確認
- [ ] Firewall確認
- [ ] 実環境での安定動作確認
