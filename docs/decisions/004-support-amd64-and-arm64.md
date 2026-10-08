# 004. Router Softwareをamd64 / arm64の両方へ移植できるように作る

## Status

Accepted

## Context

手持ち環境では、Windows PCがx86-64 / amd64、MacBookがARM64 / arm64である。

また、実機の候補にはARM SBCとx86-64のMini PCの両方がある（[003-hardware-later.md](003-hardware-later.md) を参照）。

## Decision

Debianはamd64 / arm64の両方で利用できるため、自作ソフトウェアも両方へ移植可能な設計にする。

```text
Router Software
      │
      ├── linux/amd64
      │       ↓
      │   x86-64環境
      │
      └── linux/arm64
              ↓
           ARM SBC
```

## Consequences

MacBookはメイン開発環境にはしないが、将来的にARM64 SBCへ移植する場合の検証環境として利用できる。

例えば、

```text
Windows
  ↓
Debian amd64 VM
  ↓
Router Software開発
  ↓
ARM64 Build
  ↓
MacBook上のDebian arm64 VM
  ↓
ARM SBC
```

のように、ARM64移植確認用として活用できる。
