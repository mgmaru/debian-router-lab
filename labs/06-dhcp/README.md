# 06 - DHCP

## Goal

Clientへ自動的に、

- IP Address
- Subnet Mask
- Default Gateway
- DNS Server

を配布する。

最初は既存ソフトを使い、仕組みを理解した後に自作を検討する。

## Notes

今はクライアント VM に IP を手で設定しているが、ルーター VM に DHCP サーバーを立てれば、クライアントに自動で IP・ゲートウェイ・DNS を配れるようになる。家庭用ルーターがやっているのはまさにこれである。

DHCP の仕組みは [dhcp.md](../../docs/concepts/dhcp.md) を参照。
