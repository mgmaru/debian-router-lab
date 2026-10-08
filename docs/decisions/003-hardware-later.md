# 003. 実機はVMでの開発・負荷測定後に選定する

## Status

Accepted

## Context

当初は、

- Raspberry Pi
- NanoPi
- N100系ミニPC
- ルーター向けSBC

などを候補として考えていた。

## Decision

最初にハードウェアを選定するのではなく、

> **まずDebian VM上でルーターを開発し、その後、必要な性能を見て実機を選ぶ**

方針とする。

## Reasons

### 1. ハードウェア購入が不要

最初から、

- Raspberry Pi
- NanoPi
- Mini PC

などを購入する必要がない。

### 2. ネットワークを壊しても戻せる

VM Snapshotを利用できるため、

- Routingを壊した
- Firewallですべて遮断した
- SSHできなくなった
- Network設定を間違えた

場合でも復旧しやすい。

### 3. 複数ネットワークを簡単に作れる

VMなら、

```text
WAN
LAN
DMZ
Server Network
Test Network
```

などを容易に追加できる。

### 4. 必要スペックを実測できる

開発後に、

```bash
top
htop
free -h
```

などを使用して負荷を測定する。

例えば、

```text
CPU Usage : 5%
RAM Usage : 300MB
```

なら、高性能な実機は不要と判断できる。

逆に、

```text
CPU Usage : 80%
RAM Usage : 1.5GB
```

なら、それに合わせて実機を選定する。

## Hardware Selection

開発・負荷測定後に実機を選定する。

候補：

- NanoPi系
- Raspberry Pi系
- ARM SBC
- N100系Mini PC
- Router Appliance

### 実機選定時の主な評価項目

| 項目 | 内容 |
|---|---|
| CPU | VMでのCPU使用率を基準に決定 |
| RAM | 実測値＋余裕を持たせる |
| NIC | 最低2ポート |
| NIC速度 | 1GbE / 2.5GbE |
| Architecture | ARM64 / x86-64 |
| Debian対応 | 必須 |
| Storage | microSD / eMMC / SSD |
| 消費電力 | 常時稼働を考慮 |
| サイズ | 小型を優先 |
| 冷却 | ファンレスが理想 |
