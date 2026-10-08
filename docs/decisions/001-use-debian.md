# 001. Debianを採用する

## Status

Accepted

## Context

本プロジェクトは、**ルーターの仕組みを理解すること**を目的としている。

OSそのものは自作せず、Linuxを土台として利用し、その上でルーターに必要な機能を段階的に構築・実装する。

## Decision

ルーターのOSとして **Debian** を採用する。

ルーター専用OSは使用しない。

## Reasons

- シンプルなLinux環境を構築しやすい
- Linux標準のネットワーク機能を直接学びやすい
- ルーター専用OSによる抽象化が少ない
- `iproute2`、`nftables`、`systemd-networkd` などを直接扱える
- ARM64 / x86-64 の両方で利用しやすい
- 将来的にSBCやミニPCへ移植しやすい

## Alternatives

| OS | 特徴 | 今回採用しない理由 |
|---|---|---|
| Ubuntu Server | Debian系で扱いやすい | Netplanなどの追加レイヤーを減らしたい |
| OpenWrt | ルーター用途に特化 | ルーター機能が最初から用意されすぎている |
| VyOS | 業務用ルーターに近い | 「Linuxをルーター化する過程」を学びにくい |
| pfSense / OPNsense | Firewall用途に強い | BSD系で、今回のLinux学習目的とは少し異なる |
