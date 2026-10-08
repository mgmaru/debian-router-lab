# Decisions

重要な技術選定や方針を記録する。

1つのADRには1つの決定だけを記録する。

## ADR一覧

| No. | 決定 |
|---|---|
| [001](001-use-debian.md) | Debianを採用する |
| [002](002-use-vmware.md) | Windows 11 + VMware Workstationで開発環境を構築する |
| [003](003-hardware-later.md) | 実機はVMでの開発・負荷測定後に選定する |
| [004](004-support-amd64-and-arm64.md) | Router Softwareをamd64 / arm64の両方へ移植できるように作る |
| [005](005-ssh-via-jump-host.md) | VMへのSSHは踏み台経由で行う |
| [006](006-use-ifupdown.md) | ネットワーク設定はifupdownで管理する |

---

## 現時点で決定したこと

| 項目 | 決定内容 | 記録 |
|---|---|---|
| プロジェクト目的 | ルーターの仕組みを理解する | [README](../../README.md) |
| OS自作 | しない | [README](../../README.md) |
| OS | **Debian** | [001](001-use-debian.md) |
| ルーター専用OS | 使用しない | [001](001-use-debian.md) |
| 開発マシン | **Windows PC** | [002](002-use-vmware.md) |
| 開発マシン選定理由 | VMware WorkstationでVM・仮想NIC・仮想ネットワークを構築しやすいため | [002](002-use-vmware.md) |
| CPUアーキテクチャ | 現段階では開発マシンの主要な選定理由にしない | [002](002-use-vmware.md) |
| Router Softwareの対応Architecture | amd64 / arm64 の両方へ移植可能にする | [004](004-support-amd64-and-arm64.md) |
| MacBookの位置付け | 将来のARM64移植・検証用として利用可能 | [004](004-support-amd64-and-arm64.md) |
| 開発開始環境 | **Windows 11 + VMware Workstation** | [002](002-use-vmware.md) |
| WSL2の位置付け | VM構築には使用せず、普段のLinux開発・補助作業に使用 | [002](002-use-vmware.md) |
| 初期VM台数 | **2台（Debian Router VM + Client VM）** | [overview](../architecture/overview.md) |
| Router VM NIC | **2つ（WAN用 + LAN用）** | [overview](../architecture/overview.md) |
| Client VM NIC | **1つ（LAN用）** | [overview](../architecture/overview.md) |
| Router VMの役割 | ルーター機能を設定・実装し、内部を観察する実験対象 | [overview](../architecture/overview.md) |
| Client VMの役割 | 通信を発生させ、Router VMの動作を確認する端末 | [overview](../architecture/overview.md) |
| 学習方法 | Router VMとClient VMをセットで使用する | [overview](../architecture/overview.md) |
| VMへのSSH | Windows、Router VMを踏み台にした多段SSH | [005](005-ssh-via-jump-host.md) |
| VMのネットワーク設定 | ifupdown（`/etc/network/interfaces`）で管理する | [006](006-use-ifupdown.md) |
| 最初の開発 | Linuxを手動でルーター化 | [README](../../README.md) |
| 自作Software | Linux機能を理解した後に開発 | [routerd](../../src/routerd/README.md) |
| Hardware購入 | **現時点では行わない** | [003](003-hardware-later.md) |
| Hardware選定 | VMでの開発・負荷測定後 | [003](003-hardware-later.md) |
| 実機 | 小型SBC / Mini PCを候補とする | [003](003-hardware-later.md) |
| 最終目標 | VMで作ったRouter Softwareを実機へ移植 | [README](../../README.md) |
