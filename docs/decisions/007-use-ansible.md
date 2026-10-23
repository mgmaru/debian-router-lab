# 007. VMの設定自動化はAnsibleで行う

## Status

Accepted

## Context

Phase 2〜4 では、Router VM / Client VM に対して、Network / sysctl / nftables / DHCP / DNS などを手作業で設定していく。

手作業の設定には以下の問題がある。

- VMを作り直すと、同じ設定をもう一度手で行う必要がある
- どの設定が最終的に必要だったのか、後から分かりにくくなる
- Phase 8 で実機へ移植するとき、VMと同じ状態を再現できる保証がない

そのため、Phase 5 で手動設定を自動化する。

自動化の方法には以下の候補がある。

| 方法 | 特徴 |
|---|---|
| Shell Script | 追加の道具が不要。ただし「すでに設定済みなら何もしない」処理を自分で書く必要がある |
| Ansible | SSHで接続して設定を反映する。何度実行しても同じ状態になる（冪等性）書き方がしやすい |
| Terraform | VMやクラウド上の資源そのものを作る道具。VMware Workstationには公式のProviderがない |

また、転職先としてインフラエンジニアを希望しており、実務でよく使われる構成管理ツールを経験しておきたい。

## Decision

Router VM / Client VM の **OS設定の自動化は Ansible で行う**。

- Control Node（Ansibleを実行する側）は MacBook とする
- VMへの接続は [005](005-ssh-via-jump-host.md) の多段SSHをそのまま使う
- 各機能は **手作業で設定・理解してから** Ansible に書き起こす

VMそのものの作成（VMware Workstation上でのVM作成・仮想NIC追加）は、自動化の対象にしない。

## Reasons

### Ansibleを使う理由

- Network / sysctl / nftables / DHCP / DNS の設定ファイル配布とパッケージ導入は、Ansibleが得意とする作業である
- `configs/` に置いた設定ファイルを、そのままAnsibleでVMへ配布できる
- Inventoryを変えるだけで、VM用の設定を実機へ反映できる（Phase 8 で使える）
- VMへSSHで接続できれば使えるため、VM側に専用のAgentを入れなくてよい
- 実務でよく使われており、転職活動で説明できる経験になる

### 手作業の後に自動化する理由

このリポジトリの目的は、Linuxがルーターとして動作する仕組みを理解することである。

最初からAnsibleで設定すると、「Playbookを書いたら動いた」で止まり、`ip route` や nftables の動きを理解しないまま進んでしまう。

そのため、各機能は

```text
手作業で設定
   ↓
docs/ に記録
   ↓
Ansibleに書き起こす
```

の順で進める。

### Terraformを使わない理由

- VMware Workstationには公式のTerraform Providerがなく、コミュニティ製のProvider（`vmrest` 経由）に頼ることになる
- VMの作成はVMwareの画面から行っても手間が小さく、自動化の効果が小さい
- Terraformは、AWSなどのクラウド上で使う方が実務に近い経験になる

## Consequences

- MacBookにAnsibleをインストールする必要がある
- AnsibleはVM側でPythonを使うため、Router VM / Client VMにPython 3が入っている必要がある
- Client VMはRouter VMを経由しないとInternetへ出られないため、Client VMへのパッケージ導入はNAT設定（Phase 3）の後になる
- Phase 6 の `routerd` とAnsibleの担当範囲が重なる可能性があるため、`routerd` の責務を決めるときに境界を決める

## Notes

Terraformは、このリポジトリとは別に、AWS上でこのラボと同じ構成（VPC / Subnet / Route Table / NAT Gateway / Security Group）を作る形で学習することを検討する。
