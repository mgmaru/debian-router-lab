# 002. Windows 11 + VMware Workstationで開発環境を構築する

## Status

Accepted

## Context

本プロジェクトの初期段階では、

- Routing
- NAT
- Firewall
- DHCP / DNS
- Linux Network Stack
- 自作Router Software

などを学習・実装する。

そのため開発マシンに重要なのは、**VM環境と仮想ネットワークを構築しやすいこと**である。

## Decision

**Windows PCをメイン開発マシンとして使用し、Windows 11上のVMware Workstationで開発用のVMを構築する。**

WSL2の中にVMは作らない。WSL2は普段のLinux開発・補助作業に使用する。

## Reasons

WindowsではVMware Workstationを利用し、

- 複数のVM
- 複数の仮想NIC
- NAT Network
- Host-only Network
- Custom Network

などを比較的柔軟に構成できる。

本プロジェクトでは、Router VMにWAN / LANの2つの仮想NICを持たせ、LAN側にClient VMを接続する構成が必要になる。
将来的には、DMZやLabなど複数ネットワークへ拡張する可能性もある（[network-topology.md](../architecture/network-topology.md) を参照）。

このようなネットワーク構成を作りやすいため、Windows + VMware Workstationを採用する。

## Notes

### CPUアーキテクチャは選定理由にしない

手持ち環境では、

| マシン | CPUアーキテクチャ |
|---|---|
| Windows PC | x86-64 / amd64 |
| MacBook | ARM64 / arm64 |

となる。

ただし、Windows PCを選んだ理由は、Windows PCがx86-64で、MacBookがARM64だからではない。
上記の学習・実装内容では、CPUアーキテクチャの違いは大きな選定要因にはならない。

したがって、

> 開発マシン選定ではCPUアーキテクチャよりも、VM・仮想NIC・仮想ネットワークの作りやすさを優先する。

という方針とする。

CPUアーキテクチャへの対応方針は [004-support-amd64-and-arm64.md](004-support-amd64-and-arm64.md) を参照。
