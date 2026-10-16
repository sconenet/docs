# Sconeドキュメント

> `docs/` 配下のすべてのドキュメントは**規範(NORMATIVE)**です。Rustのコードを読まずに独立した実装を可能にしなければなりません。ドキュメントとコードの不一致はバグです(信頼できる正しい情報源を特定し、修正すること)。

2つの部分から構成されます:

- **`general/`** — 仕組みの説明:フォーマットの詳細を読まなくても理解できる、プロトコルの概念とルール。
- **`technical/`** — 開発者向けの詳細仕様:正確なバイナリフォーマット、ハッシュ計算式、上限値、API、および各コンポーネントの動作。

## `general/` — 仕組み

| ドキュメント | 内容 |
|---|---|
| [architecture.md](general/architecture.md) | 権威(ブロックチェーン)/可用性(DHT)/ローカルストレージの分離、各クレートの役割、DNS解決フロー、進化可能性 |
| [naming.md](general/naming.md) | 名前付けルール:LDHアルファベット、TLD、ドメインとサブドメイン、正規化、`DomainId` の導出と16進表現 |

## `technical/` — 開発者向け仕様

| ドキュメント | 内容 |
|---|---|
| [protocol.md](technical/protocol.md) | 正準ワイヤーバイナリフォーマット:LEB128可変長整数、識別子、DNSレコード、ブロック、P2Pメッセージ、制限、バージョニング |
| [transactions.md](technical/transactions.md) | 署名済みトランザクションフォーマットv1:フィールド、ワイヤー順序(RegisterDomainは名前を運ぶ、任意の判別子)、所有者の導出、署名対象ペイロード `SCONE-TX-SIG-V1`、検証ルール |
| [blockchain.md](technical/blockchain.md) | ブロックチェーン層:`TxId`、Merkleツリー、`BlockHash`、ジェネシス、正準状態、ブロック検証、コンセンサス抽象化 |
| [keystore.md](technical/keystore.md) | `.sconekey` キーファイルフォーマット(Argon2id + XChaCha20-Poly1305)、`scone-keystore` API、`identity` コマンド |
| [storage.md](technical/storage.md) | ローカルredb永続化:テーブルスキーマ、アトミックなデルタ、メモリポリシー、信頼モデル、DoS上限 |
| [relay.md](technical/relay.md) | ネットワークリレー(`scone-network`):libp2pプロトコル、ピア発見(シード、ディレクトリ、mDNS、UPnP)、到達可能性(AutoNAT、circuit-relay、DCUtR)、メモリプール、同期、Kademlia DHT、ローカルトポロジー、Prometheusメトリクス、DNSスナップショット、完全なCLI、`.env` 設定 |
| [dns.md](technical/dns.md) | UDP + TCP DNSサーバー(M6/M7b):権威DNSスナップショット、16分割シャード化ホットパスキャッシュ(共有エントリ、アロケーションなしルックアップ、パックされたレコード、名前圧縮)、RFC 1035コーデック、TCP RFC 7766(512バイトUDP / 64 KiB TCPバジェット、TCビット)、DHTネガティブキャッシュ、再帰フォールバック、上限値 |
| [cli.md](technical/cli.md) | `scone` バイナリ:ロギングと詳細度(stdout/stderr、`-v` レベル、`RUST_LOG`)、`.env` による環境設定(優先順位:フラグ > `.env` > デフォルト) |
| [security-model.md](technical/security-model.md) | セキュリティモデル:委員会選出、シードのアンチグラインディング(分析:VRFは不要)、等価性違反(イコケーション)とスラッシング |
| [audit-methodologie.md](technical/audit-methodologie.md) | セキュリティ監査方法論:不一致、信頼できる正しい情報源、修正パス |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | Proof-of-service RFC:オペレーターのハートビート、操作の委任、最新のシート間での委員会選出 |

## 相互参照の規約

内部ドキュメントとRustコードのコメントは、リポジトリルートからの絶対パス
(例: `/docs/en/technical/protocol.md`)でこれらのページを参照します。
