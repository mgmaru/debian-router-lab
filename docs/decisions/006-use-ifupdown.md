# 006. ネットワーク設定はifupdownで管理する

## Status

Accepted

## Context

Debian でネットワークを管理する仕組みは主に 3 つある。

| 仕組み | 設定場所 | よく使われる環境 |
|---|---|---|
| ifupdown（`networking` サービス） | `/etc/network/interfaces` | 最小構成のサーバー |
| NetworkManager | `nmcli` コマンド | デスクトップ環境入り |
| systemd-networkd | `/etc/systemd/network/*.network` | 一部のクラウド・コンテナ環境 |

Router VM / Client VM はデスクトップ環境なしの最小構成でインストールしており、インストール直後の時点で ens33 は ifupdown（`/etc/network/interfaces`）で管理されていた。

## Decision

Router VM / Client VM のネットワーク設定は、**ifupdown（`/etc/network/interfaces`）で管理する**。

同じ NIC を複数の仕組みで管理しない。

## Reasons

- インストール直後から ifupdown が ens33 を管理しており、同じ仕組みに揃えられる
- 同じ NIC を複数の仕組みで管理すると、設定が取り合いになって、意図しない IP になったり、IP が消えたりする

## Notes

[001-use-debian.md](001-use-debian.md) では、Debian を選んだ理由の 1 つに `systemd-networkd` などを直接扱えることを挙げている。
今回は、インストール直後から使われている ifupdown をそのまま使う。

設定手順は [environment/debian/network.md](../../environment/debian/network.md)、設定ファイルは [configs/network](../../configs/network/README.md) を参照。
