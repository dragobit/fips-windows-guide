# FIPS v0.5.0 Windows セットアップガイド

[jmcorgan/fips](https://github.com/jmcorgan/fips) (Free Internetworking Peering System) v0.5.0 を **Windows** に導入し、**ローカルネットワーク (LAN)** と **グローバルネットワーク (インターネット越し)** の両方で `<npub>.fips` アドレスによる通信を可能にするまでの全工程をまとめた日本語ドキュメントです。

> **名称の注意**: 正しい DNS サフィックスは `.fips` です(`.fip` ではありません)。`<npub>.fips` の形で相手ノードを指定します。

## FIPS とは(30秒版)

- Nostr の鍵ペア (secp256k1 / npub) をノード ID とする、自己組織化・暗号化メッシュネットワーク。
- 中央サーバー不要。Noise IK(リンク間)+ Noise XK(エンドツーエンド)の二重暗号化。
- TUN アダプタが各 npub を `fd00::/8` の IPv6 アドレスに写像するため、SSH・HTTP・ping など既存の IPv6 対応ソフトがそのまま動く。
- ピア発見は mDNS(LAN 内)と Nostr リレー経由(グローバル、NAT 越え対応)の両方に対応。
- Windows では wintun ドライバ + Windows サービスとして動作。UDP / TCP / Tor / Nym トランスポートが使える(Ethernet / BLE は Windows 非対応)。

## ドキュメントの3レベル

| レベル | 対象 | 内容 |
|--------|------|------|
| [Level 1: 最小構成(クイックスタート)](docs/LEVEL1-quickstart.md) | とにかく動かしたい人 | インストール → テストメッシュ参加 → `<npub>.fips` で ping までの最短手順。所要約15分 |
| [Level 2: 標準構成(推奨セットアップ)](docs/LEVEL2-standard.md) | 実運用したい人 | 永続 ID、LAN(mDNS)+ グローバル(Nostr/NAT 越え)の両対応、`.fips` の全アプリ名前解決、FW 設定、疎通検証 |
| [Level 3: 詳細リファレンス](docs/LEVEL3-reference.md) | 深く理解したい人 | fips.yaml 全設定、トランスポート/NAT 越え/DNS の内部動作、ACL・セキュリティ、Windows 固有の差異と制限、トラブルシューティング |

## 前提条件

- Windows 10 / 11 (x86_64)
- 管理者権限(TUN デバイス作成とサービス登録に必要)
- IPv6 プロトコルが有効なこと(`fd00::/8` 経路を使うため)
- [wintun.dll](https://www.wintun.net/)(FIPS パッケージには同梱されないため別途入手)

## 前提知識:2つの「ネットワーク」

FIPS では「ローカル」と「グローバル」は排他ではなく、同じデーモン上で同時に動きます。

- **LAN(ローカル)**: `node.rendezvous.lan.enabled: true` で mDNS ブロードキャストによる同一 LAN 内の自動ピア発見。リレー・STUN・設定不要でサブ秒ペアリング。
- **グローバル**: 静的ピア(`peers:` に npub + アドレスを直書き)または **Nostr リレー仲介ディスカバリ**(`node.rendezvous.nostr`)で、NAT の内側のノード同士でも STUN + ホールパンチングで直接 UDP リンクを確立。

どちらの経路で繋がっても、通信はすべて `<npub>.fips` → `fd00::/8` IPv6 で統一されます。

## 情報源

本ガイドは [jmcorgan/fips](https://github.com/jmcorgan/fips) の `v0.5.0` タグ時点のドキュメント(`docs/`)・パッケージスクリプト(`packaging/windows/`)・ソースコードをもとに作成しています。最新情報・バグ修正については上流を参照してください。

> **補足**: 2026-09-06 に v0.5.1 が出ています(Linux パッケージの GLIBC 不具合修正が主目的で、Windows ユーザーへの影響は軽微ですがディスカバリ修正2件を含みます)。本ガイドの手順は v0.5.0 / v0.5.1 共通です。
