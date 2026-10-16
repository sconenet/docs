# アーキテクチャ

## まず平易な言葉で

5つの概念を理解すれば、他のすべてを理解できます:

1. **名前** — インターネット上の住所。`mysite.scone` のようなもの。
2. **あなたの24語のフレーズ** — あなたの識別情報。名前があなたのものであることを証明する権利証であり、あなたが行うすべての変更に署名します。
3. **公開台帳**(ブロックチェーン)— 誰が何を所有しているかの、共有された公開の帳簿。誰も制御できず、誰も書き換えられません。
4. **マイニング** — 名前を予約する方法。あなたのコンピューターが、誰にとっても公平で平等な作業を行います。順番待ちも仲介者もありません。
5. **リレー** — 台帳のコピーを保持し、質問に答えることでネットワークを運転するコンピューター。誰でも実行できます。数が多いほど、ネットワークは強くなります。

このページの残りはより深く立ち入ります。ここまでで読みやめても、プロジェクトはもう理解できています。

## 指導原理

Sconeは権威と可用性を分離します:

- **ブロックチェーン**は、所有権と変更の順序に関する信頼できる正しい情報源です。
- **DHT**は配布/レプリケーションシステムであり、権威では**ありません**。
- **ローカルストレージ**は、ノードの状態、インデックス、キャッシュを保持します。

ノードは、DHTから来るあらゆるデータを潜在的に悪意のあるものとして扱わなければなりません。暗号学的検証(所有者署名、チェーンの所有権/シーケンス/record_hash との照合)を経て初めて受理されます。

## 責任の分離

```text
BLOCKCHAIN
──────────
Ownership
Registration
Authorization
Ordering
Sequence
Record hash
Consensus information

DHT
───
DNS records
Signed records
Replication
Availability

LOCAL STORAGE
─────────────
Local state
Indexes
Cached DHT records
Blockchain state
```

## クレートと依存関係

```text
scone-crypto     ← 最下層:バイト列に対する純粋なプリミティブ
    ▲
scone-core       ← プロトコル型(scone-crypto に依存)
    ▲
scone-protocol  ← ワイヤー/ブロックチェーンのバイナリフォーマット
    ▲
scone-blockchain← ブロックチェーンロジック、正準状態(メモリ内)
    ▲               └ scone-topology(純粋エンジン、ネットワーク/fs/
                     async ゼロ。scone-network がデータを供給)— scone-genesis
                     (一回限りの mainnet ジェネシスツールバイナリ)
scone-storage   ← ローカル永続化:redb バックエンド + 検証済みスナップショット
    ▲
scone-network   ← p2p リレー:libp2p/QUIC、メモリプール、同期、DHT、DNS
    ▲
scone-keystore  ← 暗号化された `.sconekey` キーファイル(scone-crypto に依存)
    ▲
scone           ← バイナリ / アプリケーション
```

### 決定:`scone-core` は `scone-crypto` に依存する

`DomainId` は土台から計算できなければなりません
(`BLAKE3-256(prefix || canonical name)`)が、具体的な暗号技術がコアを汚染してはなりません。解決策:

- `scone-crypto` は最下層です。バイト列に対する純粋に決定論的な関数(キー生成用のOS RNGを加算)、コア型ゼロ、ネットワーク/ファイルシステム/async の依存ゼロ → 循環は不可能。
- `scone-core` はハッシュ計算のためだけにそれを使います。
- Ed25519の署名とキーは `scone-crypto` にあり、バイト列(`scone-protocol` が生成する正準ペイロード)に対して操作します。

### 各クレートの役割

**scone-core** — 名前(`TldName`、`DomainName`)、識別子(`DomainId`、`OwnerId`、`PublicKeyRef`)、DNSデータ(`DnsRecord`、`SignedDnsRecord`)、トランザクション(`RegisterDomain`、`UpdateDomain`)。決定論的かつポータブル。このクレートで禁止されているもの:libp2p、redb、tokio/async、ネットワーク、ファイル、OS、DNSサーバー、ブロックチェーンロジック。

**scone-crypto** — 暗号学的プリミティブ。現在:`hash256`(BLAKE3)、Ed25519の署名とキー(`SigningKey`、`PublicKey`、`Signature`、厳格なバッチ検証)、レコードグループのAEADエンベロープ(XChaCha20-Poly1305)。計画中:Merkleツリー。ルール:自作のアルゴリズムなし、監査済みの依存関係のみ。

**scone-protocol** — プロトコルのバイナリフォーマット。トランザクション、DNSレコード、ブロック(不透明なコンセンサスフィールドを持つ `Block`/`BlockHeader`)、P2Pメッセージの正準エンコード/デコード(最小限のLEB128可変長整数、裸の32バイト識別子)。アロケーションの前にサイズ制限を検査し、非正準形を拒否し、バージョニング(`PROTOCOL_VERSION`)を備えます。コンセンサス、ネットワーク、ストレージのロジックはありません。フォーマット参照: `/docs/en/technical/protocol.md`。

**scone-blockchain** — ブロックチェーンロジックと正準状態。トランザクション識別(`TxId`)、順序付きMerkleツリー、`BlockHash`、決定論的ジェネシス、ブロックとトランザクションの検証、状態適用ルール(`DomainState`/`ChainState`、メモリ内、`DomainId` による直接アクセス)、フォーク検出とブランチ選択(`fork_choice`:ファイナリティ下限 + 最長チェーン + 最小ハッシュのタイブレーク)、チェックポイントファイナリティ、そしてコンセンサス抽象化(`trait Consensus`、デフォルトでは寛容な実装)。登録PoW(マイニング/検証、ネットワークごとの難易度)は `scone-core` にあります。このクレートで禁止されているもの:ネットワーク、DHT、ストレージ(redb)、DNSサーバー。参照: `/docs/en/technical/blockchain.md`。

**scone-storage** — ローカル永続化(同梱済み)。具体的なバックエンドは `redb`(`NodeStore`)で、ブロックチェーン/DHT/DNS がredbに直接結合しないよう、最小限のトレイトの背後に隠されています。別のバックエンド(分散KV)を接続することも可能です。ページネーションされたKV状態バックエンド、ブロックログ、検証済みスナップショット(ハッシュ化されたマニフェスト + ページ)、再起動時のリストア/リプレイパスを含みます。参照: `/docs/en/technical/storage.md`。

**scone-network** — ノードのp2pリレー。libp2p/QUICスワーム、トランザクションとブロックのリレー(組織的なローカル検証)、有界のメモリプール、有界のブロック同期(128ブロック/リクエスト)、`SignedDnsRecord` のKademlia DHT、UDP+TCP DNSサーバー(GeoDNS常時有効 — 組み込みGeoIPテーブル、再帰フォールバック)、アンカーのチェックポイント/ハートビートループ(CLIはリレーと直接p2p QUICで対話し、ローカルRPCはもうありません)。到達可能性と発見:AutoNAT v2(reachable/NATedの判定)、NAT化ノード用のcircuit-relay v2 + DCUtR、mDNS(ゼロ設定のLAN発見、devnetではデフォルトでON)、UPnP(ルーターによるQUICポートマッピング、オプトイン)。`.env` ファイルによる設定(`.env.example` を参照)。参照: `/docs/en/technical/relay.md`。

**scone-keystore** — Ed25519署名キーの暗号化された保存(`.sconekey` ファイル:Argon2id + XChaCha20-Poly1305)。このクレートがファイルシステムを所有するため、`scone-crypto` は純粋なまま保てます。参照: `/docs/en/technical/keystore.md`。

**scone-topology** — ローカルp2pトポロジー最適化のための純粋(PURE)エンジン。ネットワークゼロ、ファイルシステムゼロ、asyncゼロ。libp2pもスワームも知りません。`scone-network` が観測(ping RTT、接続)を供給し、近傍管理の決定を受け取ります。

**scone-genesis** — mainnetジェネシス準備のための一回限りのツールバイナリ(キックスターター)。バージョン付きJSONマニフェスト、アロケーション、決定論的ブロック。

**scone** — バイナリ/CLI。実際に使うクレートにのみ依存します(現在:リレー用のscone-network、scone-blockchain、scone-storage、scone-keystore、scone-protocol、scone-core、scone-crypto、scone-topology)。

## ノード

Sconeノードは次を組み合わせます:

1. ブロックチェーン(コンセンサス検証、所有権状態)。
2. DHTルーティング(`SignedDnsRecord` の公開/取得)。
3. ローカルストレージ(状態、インデックス、キャッシュ)。
4. 先頭に立つリゾルバー / DNSサーバー。

### 新しいノードの起動(状態ブートストラップ、P0.3/P0.4)

新しいノードはジェネシスをリプレイしません。検証済みのファイナライズされた状態から始まります。

```text
finalized checkpoint (committee, signature quorum — validated by
        │              the chain layer before any use)
        ▼
verified snapshot (manifest + paginated pages; the importer
        │              RECOMPUTES the SMT root from the entries
        │              and compares it to the signed state_root —
        │              a falsified/incomplete snapshot is rejected
        │              before any write)
        ▼
anchor block H (requested from the network BY HASH)
        │
        ▼
recent blocks H+1..H+k (relay, increasing height)
        │
        ▼
synced — at boot, replay of ONLY the suffix above H
```

2つのノードプロファイル:

- **アーカイブノード**(オペレーターフラグ):すべての履歴ブロックを保持し、ネットワークに提供し、既存のパスで起動します。
- **フル状態ノード**:Hにおける完全な状態と接尾辞を保持しますが、Hより下の履歴は保持しません。将来のブロックを検証し、状態を提供しますが、履歴は提供しません。

このパスはオプトインです。検証済みスナップショットのインポートは明示的なオペレーター操作(`import_verified_snapshot`)であり、自動化はされません。既存ノードのデフォルト動作は変わりません。規範的詳細: `/docs/en/technical/storage.md` の「VERIFIED state snapshot」節と「State bootstrap」節。

## DNS解決フロー

```text
DNS query (example.uip)
    │
    ▼
Node: DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || "example.uip")
    │
    ├── authoritative DNS snapshot (RAM) ─► unknown/expired → NXDOMAIN
    │                                          (without touching the core)
    ├── sharded DNS cache (qname,qtype) ───► hit: shared entry,
    │                                          lazy verification
    ├── local cache (SignedDnsRecord) ─────► verification (signature, owner,
    │                                          sequence, record_hash vs chain)
    ├── DHT get(DomainId) ─────────────────► same verification
    │                                          (miss → 30 s negative cache)
    ▼
DNS answer (if and only if valid)
```

DNSスナップショット(適用された各ブロックの後に公開される、すべての生存ドメインの存在 + 失効)は、存在判断がリレーのランループを通らないことを保証します。コアは、実際のレコード取得のためだけに触れられます。

## 進化可能性

- バージョン付きプリイメージ(`SCONE-DOMAIN-V1`、`SCONE-PUBKEY-V1`、`SCONE-OWNER-V1`、…)。導出ルールを変えることは、既存のものを壊すのではなく、新しい名前空間を作ることです。
- `RecordData` はバリアント + 読み取り時に未知の型のための `Unknown { type_code, data }` で拡張可能。
- ワイヤーフォーマットの破壊的変更のための `PROTOCOL_VERSION`(`scone-protocol`)。
- チェーンはハッシュのみを保存します。大きなデータはDHTに留まります。
