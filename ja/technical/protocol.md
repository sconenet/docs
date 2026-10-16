# プロトコル

Sconeプロトコルの参照文書:バイナリフォーマット、正準エンコーディング、トランザクション、DNSレコード、ブロック、P2Pメッセージ、制限、バージョン互換性。参照実装は[`scone-protocol`](../../../crates/scone-protocol)クレートです。

**（暫定）**と記された項目は最終版ではありません。

## プロトコルバージョン

- 定数: `PROTOCOL_VERSION: u32 = 1`(`scone-protocol`)。
- `Hello.version`(ハンドシェイク)と `BlockHeader.version` が運びます。
- 互換性ルール:
  - 受信バージョンがローカルバージョンより**高い** → 拒否(`ProtocolError::UnsupportedVersion`)。ノードは知らないバージョンのフォーマットを推測しません。
  - バージョン `0` → 無効。
  - フォーマットの集合は1つだけ:トランザクションのバージョンバイト(`0x01`)とブロックのバージョンは、ローカル値と**厳密に**一致しなければなりません。
- この段階でメジャー/マイナーの分割はありません:単一の整数であり、ワイヤーフォーマットの破壊的変更ごとに増加します。

## バイナリフォーマットの原則

- プロトコルデータはバイナリのみ。決してJSONにしない。
- 決定論的かつ正準:論理的に等しい2つのオブジェクトは** exactly the same bytes** を生成します。
- 逐次パース。可能な限りアロケーションなし(バイトの借用)。
- すべてのネットワーク入力は信頼できません:長さとカウンターは、いかなるアロケーションの**前**に検査されます。デコードは決してパニックしません。

### エンディアン

- すべての整数はLEB128可変長整数(7ビットごと、最下位から)。ネイティブのエンディアンは適用されません。
- IPv4/IPv6アドレスと32バイト識別子:長さプレフィックスなしの生バイト配列。
- フォーマットに固定サイズのワイド整数はありません:単一の整数エンコーディングルール。

### 可変長整数(最小LEB128)

各バイトは7ビットのデータを運びます。最終バイト以外は上位ビットが1です。`u64` は1〜10バイトに収まります。

| 値 | バイト(16進) |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16,384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

エンコーディングは**最小**でなければなりません:過剰な形式はデコード時に拒否されます(`InvalidVarint`)。これが、値がただ一つの有効バイト列を持つための必要条件です(正準エンコーディング)。拒否: `80 00`(非最小)、`ff×9 02`(u64オーバーフロー)、`ff×10`(長すぎる)。

### 文字列とバイト

- `bytes`: 可変長整数の長さ + 生バイト。
- `str`: 同じ形式で、必須のUTF-8デコード付き。
- すべての長さには上限があります([制限](#制限)を参照)。読み取り前に検査されます。

### 識別子

| 型 | ワイヤー表現 |
|---|---|
| `DomainId` | 32生バイト |
| `TldId` | 32生バイト |
| `OwnerId` | 32生バイト |
| `PublicKeyRef` | 32生バイト |
| `RecordHash` | 32生バイト |
| `BlockHash` | 32生バイト |
| `MerkleRoot` | 32生バイト |
| `DomainName` | `str`(≤ 253バイト、`scone-core` が再検証) |

導出(`BLAKE3-256` + `SCONE-DOMAIN-V1` プレフィックスなど)は `scone-core` が定義し続けます(`/docs/en/general/naming.md` 参照)。

## トランザクション

```text
Transaction = disc u8 || version u8 || payload
```

| 判別子 | 型 |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x52` | `UpdateDomain` |
| `0x93` | `RegisterTld` |

各判別子は任意の不透明定数です(論理的な並びなし。`0x00` は決して有効ではない)。完全な規範表は `/docs/en/technical/transactions.md` にあります。

`version` バイト(現在の値 `0x01`)は判別子の後に続きます。その他の値は明示的に拒否されます(`UnsupportedVersion(value)`)。**すべてのトランザクションは署名されます**(Ed25519)。完全な仕様、署名対象ペイロード、検証ルールは **`/docs/en/technical/transactions.md`**(規範)にあります。

2026年9月のワイヤー変更以降、**トランザクションは `network` フィールドを運びません**: `network_id`(`str` ≤ 16)は**署名ペイロードにのみ**注入されます。あるネットワーク向けに署名されたtxは、別のネットワークでは決して検証されません(`/docs/en/technical/blockchain.md` 参照)。

### REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

`RegisterDomain` は**正準名を平文で**運び、続いてその `DomainId` が続きます。整合性 `domain_id == DomainId(name)` はデコード時に再検査されます(再計算、決して信頼しない)。`proof` は登録PoWを運びます(20バイト: nonce + エポック + 難易度 — エポックソルト `epoch = height/144` は適用ブロックのもので、適用時にネットワーク定数とブロックのエポックに対して厳格に検証されます。不一致時は型付き拒否 `ProofEpochMismatch` — M8b、`transactions.md` 参照)。`public_key` は署名者のEd25519キー(ちょうど32バイト)。`owner` はそのキーのBLAKE3導出でなければなりません(再計算、決して信頼しない)。`signature` はちょうど64バイトです。

### REGISTER_TLD (0x93)

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

TLDのクレーム(ドメイン登録とは別のレジストリ、M7)。TLDはドメインの名前と同様に**平文で運ばれます**(M8b — PoWチャレンジは名前から導出)。`tld_id` はその名前の `SCONE-TLD-V1` 導出であり、構成上すべての `DomainId` と重なりません。状態ルール(M8b): TLDが空き + ICANN予約でない(`IcannTldReserved` — ICANNルートTLDは誰もクレームできません)+ ラベルの**長さベース**難易度のPoW(`pow_length` 曲線 — `blockchain.md` 参照)。クレームされたものはクローズ(assignのみ)。

### UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

デコード時に再検査される不変条件: `sequence > 0`、かつ `owner` == `public_key` の導出。`timestamp` なし:順序付けは `sequence` を通じて行われます。トランザクションはコンパクトな参照のみを運びます。完全なDNSコンテンツはDHTに留まります(下記参照)。

## DNSレコード(DHT側)

完全なDNSレコードは決してブロックチェーンに入りません。DHTを通じて交換され(`GetRecord`/`Record` メッセージ)、オンチェーンでは `RecordHash` のみでコミットされます。

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

型コードは**実際のDNSコード**(IANA)です。TTL(秒、可変長整数)は型コードの後に続き、レコード全体をカバーします: 境界 `MIN_TTL = 1` .. `MAX_TTL = 2,592,000`(30日。デフォルト `DEFAULT_TTL = 86,400`)。

| コード | 型 | ペイロード |
|---|---|---|
| 1 | A | 4バイト |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`, `name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16バイト |
| 33 | SRV | `priority.v`, `weight.v`, `port.v`, `target(name)` |
| 257 | CAA | `flags u8`, `tag`(1–15 B), `value`(`str` ≤ 4096) |
| 32768 | GEOROUTE | GeoRoute(下記参照) |
| その他 | Unknown | code.v + `bytes` ≤ 4096 |

例: `A 192.0.2.1`(TTL 86,400)→ `01 80 a3 05 c0 00 02 01`(型、TTL可変長整数、アドレス)。

既知のコードを運ぶ `Unknown` はエンコード時に拒否されます(デコードがそれを生成することは決してできません)。

#### GEOROUTE(GeoDNS)

Sconeのプライベート型(IANA範囲外)。1つの地理的ルーティングルール:

```text
GeoRoute  = scope || count.v(≤ 32) || endpoint…
scope     = 0x00                    -- world
          | 0x01 continent.v        -- UN M49
          | 0x02 country.v          -- ISO 3166-1 numeric
          | 0x03 region.v           -- region code
          | 0x04 asn.v              -- AS number
          | 0x05 prefix.addr[16] || prefix.len u8   -- IP prefix (normalized)
endpoint  = target || priority.v || weight.v
target    = 0x00 octets[4]          -- binary IPv4
          | 0x01 octets[16]         -- binary IPv6
```

**正準ルール**: あるスコープのエンドポイントは**ソートされて**エンコードされます(優先度昇順、次に重み降順、次にターゲット昇順)。ソートはエンコード時に適用され、未ソートのワイヤーはデコード時に拒否されます(`NonCanonical`)。レコードセットと同様です。完全に重複するエンドポイントは拒否されます。

**検証**: `priority > 0`、`weight ≤ 100`(0 = 非アクティブなエンドポイント)、スコープごとに ≤ 32エンドポイント(`GEO_MAX_ENDPOINTS`)。ローカルテーブル(`continent_of_country`)が知らない国/大陸コードは、ワイヤーレベルでは**受理されます**: リゾルバーのテーブルはパブリッシャーより新しい可能性があります。TTLは親の `RecordData` のもの(変更なし)。

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**正準ルール**: レコードセットは順序なしのコレクションです。各レコードの個別エンコーディングの辞書式順で**ソートされて**エンコードされ、厳密に増加します(重複は拒否)。未ソートのワイヤーはデコード時に拒否されます(`NonCanonical`)。元の `Vec` の順序は転送されません。

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

ハッシュは**レコードの内容のみ**をカバーします。所有者の署名は、同じ正準バイト列に対して別個に検証されます。*(決定: コミットメントは署名をカバーしない — 本文書の以前のバージョンは「完全なSignedDnsRecord」に言及していましたが、ハッシュを署名に結合しても何ももたらさず、コミットメントを署名方式に縛り付けるだけでした。)*

## ブロック

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65,536)
```

- `version`: プロトコルバージョン(上記のルール)。
- `height`: 高さ(ジェネシス = 0)。
- `prev_hash`: 前のブロックのハッシュ(ジェネシスブロックではゼロ)。
- `tx_root`: トランザクションリストへのコミットメント(`scone-blockchain` が定義するMerkleツリー。`/docs/en/technical/blockchain.md` 参照)。
- `timestamp`: 順序情報(Unix、秒)。
- `consensus`: 生成者のペイロード(`SCONE-BLOCK-V1 ‖ pk[32] ‖ sig[64]`、`blockchain.md` 参照)— ネットワークのジェネシスは生の `network_id` を運び、アロケーションジェネシス(mainnetキックスターター)は `ALLOC-V1` マニフェスト(≤ 60 KiB)を運ぶことがあります。`MAX_CONSENSUS_LEN`(64 KiB)で有界です。

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

逐次パース。カウンターはアロケーションの**前**に上限が課され、いかなる切り詰めも拒否されます。レコードセットとは異なり、トランザクションの順序は**意味を持ち**(コンセンサスの順序)、厳密に保存されます。

`BlockHash`(正準ヘッダーのハッシュ)と `tx_root` は、ここで定義されるエンコーディング上で `scone-blockchain` によって計算されます(正確な計算式は `/docs/en/technical/blockchain.md`)。

## P2Pメッセージ

```text
Message = disc u8 || payload
```

トランスポート(フレーミング、libp2pなど)はスコープ外です。トランスポートフレームは `MAX_MESSAGE_LEN`(1 MiB)を超えるべきではありません。

| 判別子 | メッセージ | 方向 | ペイロード | 制限 | 期待される応答 |
|---|---|---|---|---|---|
| `0x01` | `Hello` | 双方 | `version.v` | — | `Hello` |
| `0x02` | `Ping` | 双方 | — | — | `Pong` |
| `0x03` | `Pong` | 双方 | — | — | — |
| `0x04` | `GetBlock` | 双方 | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | 双方 | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | 双方 | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | 双方 | `Transaction` | — | — |
| `0x08` | `GetRecord` | 双方 | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | 双方 | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | 双方 | `Checkpoint` | 署名者 ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | 双方 | — | — | チェックポイントウィンドウ |
| `0x0c` | `GetStatus` | 双方 | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | 双方 | ノード状態(ネットワーク、チップ、高さ、NAT、ドメイン、メモリプール、ピア、ファイナリティ、新鮮なオペレーター) | — | — |
| `0x0e` | `LookupName` | 双方 | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | 双方 | 名前 + TLDのオンチェーン状態 | — | — |
| `0x10` | `GetOwnedNames` | 双方 | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | 双方 | 所有者の生存ドメイン + TLD | — | — |
| `0x12` | `Error` | 応答 | アプリケーションレベルのエラーメッセージ(`str`) | — | — |

### チェックポイント(M5)

集約チェックポイントはワイヤー上で次のように運ばれます: タグ `SCONE-CKPT-V1` || 本体 `CheckpointData`(116 B、LE — 署名プリイメージの末尾と同一: `epoch(8) height(8) block_hash(32) prev_checkpoint_hash(32) state_root(32) recovery(4)`)|| `count.v` || 署名者ごとの `pk[32] ‖ sig[64]`。**pk昇順にソート、重複なし**、`count ≤ 64`(`MAX_CHECKPOINT_SIGNERS`)。厳格なデコード: 切り詰め、誤ったタグ、非正準pk、重複または未ソート = 拒否(`None`)。決してパニックしません(破損ファズテスト済み)。

`GetCheckpoints` はファイナリティウィンドウを要求します。応答は、1つ以上の `Checkpoint` メッセージによる `checkpoint_window()` ウィンドウ(最大 `CHECKPOINT_KEEP` チェックポイント、古いものから新しいものへ)です。

`TxId` は現在 `scone-blockchain` が定義します
(`BLAKE3-256("SCONE-TX-V1" || canonical(Transaction))`、`/docs/en/technical/blockchain.md` 参照)。メモリプールとウィンドウ化アンチリプレイ(`TxReplay`、`REPLAY_WINDOW_BLOCKS = 256`)はリレーに実装済み(M4)。`GetTransaction` のみ引き続き延期されています(その定義は残りのコンセンサス決定に依存します)。

## ブロックチェーン / DHT の分担

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (full, signed)
sequence
record_hash
```

チェーンは*誰が*所有し、*現在の状態が何か*(sequence + record_hash)を宣言します。DHTは*コンテンツ*を提供します。DHTレコードが受理されるのは、次の場合のみです:

1. その署名が、チェーンに登録された所有者に対して有効であること。
2. その `sequence` がチェーンのものと整合する(またはそれより高い)こと。
3. 正準エンコーディング上で再計算された `RecordHash` が、チェーンの `record_hash` と一致すること。

## 制限

| 定数 | 値 | 役割 |
|---|---|---|
| `MAX_NAME_LEN` | 253 | 名前の長さ |
| `MAX_TLD_LEN` | 63 | TLDの長さ |
| `MAX_TXT_LEN` | 4096 | TXT文字列 |
| `MAX_PROOF_LEN` | 256 | 登録プルーフ |
| `MAX_SIGNATURE_LEN` | 128 | 署名(Ed25519 = 64) |
| `MAX_RECORDS_PER_SET` | 256 | セットあたりのレコード数 |
| `MAX_DHT_RECORD_LEN` | 64 KiB | エンコード済み署名DNSレコード(DHTペイロード: ワイヤー、ストレージ、gossip)— 合計上限はエンコード時とデコード前に検査。現実的なセットの15〜30倍上(SPF+DKIM+DMARC ≈ 2〜5 KiB)、トランスポート上限の16分の1。 |
| `GEO_MAX_ENDPOINTS` | 32 | GeoRouteスコープあたりのエンドポイント数 |
| `MAX_UNKNOWN_DATA` | 4096 | 未知の型のデータ |
| `MAX_TXS_PER_BLOCK` | 4096 | ブロックあたりのトランザクション数 |
| `MAX_CONSENSUS_LEN` | 64 KiB | ヘッダーのコンセンサスぺイロード(`ALLOC-V1` ジェネシスマニフェストのために拡張 — mainnetキックスターター) |
| `MAX_BLOCKS_PER_REQUEST` | 128 | `GetBlocks` あたりのブロック数 |
| `MAX_MESSAGE_LEN` | 1 MiB | トランスポートフレーム(推奨) |

超過はすべて `LimitExceeded`。アロケーションの前に検出されます。悪意あるピアが、デコード時に不釣り合いなアロケーションを引き起こすことはできません。

## 正準エンコーディング

基本ルール:

```text
canonical object → canonical binary encoding → hash / signature
```

- 最小可変長整数のみ。
- レコードセットは厳密増加でソート。
- 論理的に等しいオブジェクトは、有効なエンコーディングをちょうど**1つ**だけ持ちます。
- デコードは非正準形を拒否します(`NonCanonical`)。同じオブジェクトに対する複数の有効エンコーディングは存在しません。

## エラー処理

`ProtocolError`(`scone-protocol::error`):

| バリアント | 意味 |
|---|---|
| `Truncated` | 値の途中で入力が終了 |
| `InvalidVarint` | 非最小、オーバーフロー、または長すぎる可変長整数 |
| `IntegerOutOfRange` | 整数がターゲット型に収まらない |
| `UnsupportedVersion` | ゼロまたはローカルより高いバージョン |
| `UnknownDiscriminant` | 未知のトランザクション/メッセージ判別子 |
| `LimitExceeded` | 制限を超える長さやカウント |
| `InvalidUtf8` | UTF-8でない文字列 |
| `TrailingBytes` | 完全な値の後に残ったバイト |
| `Validation` | `scone-core` の不変条件違反 |
| `NonCanonical` | フィールドごとには有効だが非正準なエンコーディング |

デコードはネットワークデータに対して決してパニックしません。

## 署名

- 方式: `scone-crypto` 経由の **Ed25519**(RFC 8032)
  (`ed25519-dalek`、`verify_strict` 検証 — なよなよした署名と弱いキーは拒否)。
- **チェーントランザクション**: すべての `RegisterDomain` / `RegisterTld` / `UpdateDomain` は、ペイロード `"SCONE-TX-SIG-V1" || canonical_encode(tx_without_signature)` に対する `public_key`(32 B)+ `signature`(64 B)を埋め込みます。完全な規範仕様: `/docs/en/technical/transactions.md`。
- **DHTレコード**(`SignedDnsRecord`): 署名対象ペイロードは `DnsRecord` の正準エンコーディングです。検証は `record + owner + signature` をブロックチェーンに登録された識別情報とクロスさせます。
- 横断的ルール: `owner` とすべてのハッシュ/コミットメントは**再計算**されます。提供された値から取ることは決してありません。

## コンセンサスルール

実装済み(`/docs/en/technical/blockchain.md` に記述): 署名済みブロック生成者(`SCONE-BLOCK-V1` ペイロード、チェックポイント選出の委員会)、集約チェックポイントによるファイナリティ(`SCONE-CKPT-V1`、ファイナリティ下限でのフォーク選択)、名前の長さによる登録PoW。残るトピック: 細粒度のアンチスパム手数料、所有者キーのローテーション。
