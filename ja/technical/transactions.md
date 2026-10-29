# 署名済みトランザクション(フォーマットv1)

署名済みトランザクションフォーマットの**規範**参照文書。
フィールド、正確なワイヤー順序、サイズ、署名対象ペイロード、検証ルールを規定し、Rustを読まずに独立した実装を可能にします。参照実装は [`scone-protocol`](../../../crates/scone-protocol)(エンコーディング)と [`scone-blockchain`](../../../crates/scone-blockchain)(検証)です。

`PROTOCOL_VERSION` = **1**。フォーマットの集合は1つだけです: `0x01`(トランザクション内)以外のバージョンバイト、またはローカルバージョンと異なる値(ブロック内)は `UnsupportedVersion` で拒否されます。

## 概要

```text
Transaction = disc u8 || version u8 || payload
```

| フィールド | サイズ | 役割 |
|---|---|---|
| `disc` | 1 B | トランザクション型(下表) |
| `version` | 1 B | **厳密に `0x01`**(txフォーマットバージョン) |

**トランザクションはワイヤー上でもはや `network` フィールドを運びません**(2026年9月の変更): `network_id`(`str` ≤ 16、ASCII `[a-z0-9-]` — 例: `scone-testnet`、13バイト)は**署名ペイロード内にのみ**存在します — 署名者が最初のフィールドとしてそこに注入します([署名対象ペイロード](#署名対象ペイロード)参照)— PoWダイジェスト内でもネットワークごとに留まります。したがって、あるネットワーク向けに**署名された**トランザクションは、別のネットワークでは決して検証されません: 自分のネットワークを注入するチェーンが再計算したペイロードに対して、その署名は失敗します(`InvalidSignature`。`/docs/en/technical/blockchain.md` — ネットワークとジェネシス参照)。

### 判別子(規範表)

各判別子は**固定された任意の不透明定数**です。値は意図的に異なるものが選ばれており、ワイヤーが型間の論理的な順序を決して示唆しないようにしています(0x01/0x02/0x03… の並びなし)。`0x00` は**決して**有効な型ではありません(典型的な破損バイト)。将来の型は、この表で一度だけ文書化される、未使用の空き値を取ります。

| 判別子 | 型 | 役割 |
|---|---|---|
| `0x21` | `RegisterDomain` | ドメインのクレーム |
| `0x52` | `UpdateDomain` | ドメインのDNSデータの新バージョン |
| `0x93` | `RegisterTld` | トップレベルドメインのクレーム(M7) |
| `0x47` | `TransferTld` | TLDの新しい所有者への移転(M8a。カウンター紐付けの譲渡PoW) |
| `0x6B` | `RevokeTld` | TLDの放棄(M8a) |
| `0xB8` | `SetTldOpen` | セルフ登録へのTLDの**不可逆**オープン化(一方向、owner-option。`open=false` は型付き拒否) |
| `0xD4` | `AssignDomain` | TLD所有者によるドメインの直接割り当て(M8a) |
| `0x3C` | `RenewDomain` | ドメイン登録の延長(M8a。owner-option以降、正準名を運ぶ — オープンTLDは失効に追従) |
| `0x8F` | `RenewTld` | プライベートTLDの3年期間の更新(生成と同等のPoW) |
| `0x15` | `SlashTx` | アンカーの等価性違反の証拠 + 追放(M9) |
| `0xE2` | `TransferDomain` | ドメインの新しい所有者への移転(M8c。カウンター紐付けの譲渡PoW) |
| `0x71` | `Heartbeat` | オペレーターの生存証明(proof-of-service RFC) |
| `0xA9` | `SetTldOperator` | プライベートTLDの運用の委任(proof-of-service RFC。カウンター紐付け) |
| `0x5E` | `SetDomainOperator` | ドメインの運用の委任(proof-of-service RFC。カウンター紐付け) |
| `0x66` | `SetRecordDelegates` | ドメインのレコードデリゲートキー — KSK/ZSK-lite: これらのキー**だけ**が、所有者とともに `UpdateDomain` に署名できる(カウンター紐付け) |

`0x01` 以外のバージョンバイトは明示的に拒否されます
(`UnsupportedVersion(受信した値)`): 未知または廃止されたフォーマットが偶然パースされることは決してありません。

## SLASH_TX (0x15, M9)

アンカーが**同じエポックで2つの矛盾するチェックポイント**に署名した(等価性違反)ことのオンチェーン証拠。誰でも提出できます — 証拠は暗号学的で自己完結です:

```text
offender[32] | evidence_a (CheckpointData 116 B)
  | sig_a[64] | evidence_b (CheckpointData) | sig_b[64]
  | reporter_pk[32] | reporter_signature[64]
```

- `validate()` は構造を検査します: 同じエポック、異なる `signing_hash`、2つの異なる署名、正準ペイロードに対する報告者の署名。
- 状態ルール(ジャーナル記録付き、リオルグで可逆): 告発された者の両方の署名が、同じエポックの2つの異なる署名ハッシュに対して検証されなければなりません。告発された者は適格プールにいなければなりません(そうでなければ `SlashOffenderNotInPool`)。続いて**追放** — キーはPoSプールから永続的に除去されます(状態内のブラックリスト)。
- 最小限の制裁: 告発された者のドメイン/TLDは没収され**ません**(文書化されたM9の決定)。
- 同じチェックポイントの2回 = 等価性違反ではなく(拒否)、異なるエポック = 拒否、偽造署名 = 拒否。

## REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

| フィールド | 型 | 制約 |
|---|---|---|
| `name` | `str` ≤ 253 | クレームされるドメインの正準名。**平文で運ばれる**(下記参照) |
| `domain_id` | 32生バイト | `DomainId(name)` と等しくなければならない(MUST)(再計算、決して信頼しない) |
| `owner` | 32生バイト | `public_key` の導出でなければならない(MUST)(下記参照) |
| `timestamp` | 可変長整数 | 順序情報(Unix、秒) |
| `proof` | `bytes` ≤ 256 | 登録PoWペイロード([下記](#登録プルーフ・オブ・ワークpow)参照) |
| `public_key` | 32生バイト | 署名者のEd25519公開鍵(RFC 8032) |
| `signature` | **ちょうど64バイト** | 署名対象ペイロードのEd25519署名 |

### 名前はトランザクションが運ぶ

`RegisterDomain` は正準名を平文で運び、続いてその `DomainId` が続きます。整合性 `domain_id == DomainId(name)` は、エンコード、デコード、検証の各段階で再検査されます(多層防御): 運ばれた名前とIDが一致しない `RegisterDomain` は無効です。`DomainId` はコンセンサス上の識別情報のまま(チェーンとDHTのキー)であり、運ばれる名前がチェーンを自己記述的にし、署名がクレーム全体をカバーします。

## REGISTER_TLD (0x93)

トップレベルドメインのクレーム(TLDレジストリ、M7):

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) open(0x00|0x01) public_key[32] signature[64]
```

| フィールド | 型 | 制約 |
|---|---|---|
| `tld` | `str` ≤ 63 | TLDの正準名。**平文で運ばれる**(M8b: PoWチャレンジは名前から導出。`RegisterDomain` が自分の名前を運ぶのと同様) |
| `tld_id` | 32生バイト | TLDの識別情報: `BLAKE3-256("SCONE-TLD-V1" || tld)` — `tld` の導出でなければならない(MUST)(再計算) |
| `owner` | 32生バイト | `public_key` の導出 |
| `timestamp` | 可変長整数 | 順序情報(Unix、秒) |
| `proof` | `bytes` ≤ 256 | 登録PoWペイロード([下記](#登録プルーフ・オブ・ワークpow)参照) |
| `open` | `0x00\|0x01` | **オープン**誕生: TLDは直接 `TldState::Open` に着地(誰のものでもない、セルフサービス + PoW。`open_tld_pow_difficulty` レートでの単一PoW — プライベートより安い: 共有リソース、PoSシートも排他性もなし)。**プライベート**誕生: 履歴的動作(所有者、PoSシート、`tld_pow_difficulty`) |
| `public_key` | 32生バイト | 署名者のEd25519公開鍵 |
| `signature` | **ちょうど64バイト** | 署名対象ペイロードのEd25519署名 |

`TldId` 空間は `DomainId` 空間と**素**です(異なる導出プレフィックス — `/docs/en/general/naming.md` 参照): TLD `uip` の登録(`*.uip` 名前空間へのクレーム)がドメインの識別情報を横取りすることは決してなく、その逆もありません。適用時の状態ルール: TLDは空きでなければなりません(ドメインレジストリとは別のレジストリ)。

## M8aファミリー — ワイヤーフォーマット

5つの型がTLDレジストリとドメインライフサイクルを完成させます。
M8a以降の規範ワイヤーフォーマット。**完全な状態ルールはM8b以降**(`/docs/en/technical/blockchain.md` 参照): TLD所有者必須、クレームがセルフサービスであるところではどこでもPoW、ドメイン側の失効と猶予。ペイロードはワイヤー上で `network` フィールドを運びません(署名ペイロード内にのみ存在)。

### TRANSFER_TLD (0x47)

```text
tld_id[32] owner[32] new_owner[32] proof.v public_key[32] signature[64]
```

| フィールド | 型 | 制約 |
|---|---|---|
| `tld_id` | 32生バイト | 移転されるTLD |
| `owner` | 32生バイト | `public_key` の導出(現在の所有者 / 署名者) |
| `new_owner` | 32生バイト | 受取人の識別情報(不透明: どの値も構文的には有効。それを使えるキーへの結合はM8bの状態ルールです) |
| `proof` | 有界(可変長整数 ‖ バイト) | 名前空間の**現在の**カウンターバインディングに対してマイニングされたアンチチャーン譲渡プルーフ([ミューテーションカウンター](#ミューテーションカウンター)参照)、フラット難易度 `transfer_pow_difficulty`、エポックソルト |
| `public_key` | 32生バイト | 署名者のEd25519公開鍵 |
| `signature` | **ちょうど64バイト** | 署名対象ペイロードのEd25519署名 |

タイムスタンプもシーケンスもなし: TLDは同時にちょうど1人の所有者を持ちます。最初に適用された移転が勝ちます(チェーン順序のみ)。

### REVOKE_TLD (0x6B)

```text
tld_id[32] owner[32] public_key[32] signature[64]
```

TLDの自発的な放棄(その後、新しい `RegisterTld` で再クレーム可能)。現在の所有者が署名します。意図的にタイムスタンプやシーケンスなし — 取り消しは、現在の状態に対する一回きりの操作です。

### SET_TLD_OPEN (0xB8)

```text
tld_id[32] owner[32] open u8 open_proof(proof) public_key[32] signature[64]
```

| フィールド | 型 | 制約 |
|---|---|---|
| `open` | 1バイト | **厳密に `0x00` または `0x01`**(正準形。その他の値 → デコードエラー。強制変換は決してしない) |
| `open_proof` | `proof`(長さ可変長整数 + バイト) | **署名済み**フィールド。`open = 0x01`(オープン化)のとき必須かつ検証される(オープン化PoW、ネットワークの `open_tld_pow_difficulty`、チャレンジ `SCONE-TLD-OPEN-V1 ‖ tld_id`)。`open = 0x00`(クローズ化)のとき空(クローズ化は無料) |

**クローズされた**TLD = assign専用(ドメインはTLD所有者による `AssignDomain` でのみ生成)。**オープンな**TLD = 誰でも登録PoWを行い、空きドメインをクレームできます(`RegisterDomain`)。**オープン化**には追加のプルーフ・オブ・ワークがかかります(TLDクレームより軽い: mainnetで約3日対約1週間)。所有者が `SetTldOpen` で一度支払います — プルーフは署名済み `open_proof` フィールドを通り、署名後にすり替えることはできません。

### ASSIGN_DOMAIN (0xD4)

```text
name(str ≤ 253) domain_id[32] owner[32] assignee[32] proof.v public_key[32] signature[64]
```

`RegisterDomain` の鏡像: 正準名は平文で運ばれ、`domain_id` は `DomainId(name)` と等しくなければなりません(デコード、エンコード、検証時に再計算)。**名前のTLDの所有者**が署名します(適用時に検査、M8b)。`assignee` がドメインの最初の所有者になります(`new_owner` と同様に不透明)。独自のフラットなアンチスパムプルーフを運びます(`assign_pow_difficulty`、チャレンジ `SCONE-DOMAIN-ASSIGN-V1 ‖ name` — 意図的にフラット: 名前空間の排他性はTLD生成時に支払済み)。

### RENEW_DOMAIN (0x3C)

```text
name(str ≤ 253) domain_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| フィールド | 型 | 制約 |
|---|---|---|
| `name` | `str` ≤ 253 | **平文で運ばれる**正準名(owner-option: 更新は配下のオープンTLDの死を先送りするため、チェーンは名前を知らなければならない)。`domain_id` は `DomainId(name)` と等しくなければならない(再計算) |
| `valid_until` | 可変長整数 | 新しい登録失効時刻(Unix、秒) |
| `proof` | 有界 | **更新の認可**(下記参照) |

ドメインの現在の所有者が署名します。`valid_until` が実際に失効を延長するか(そして最大でも更新1期間分か)は状態ルールです(M8b)。

#### 更新の認可 — TLDに応じたPoWまたはPoS

更新のコストは、配下のTLDの性質に依存します(適用時にブロック状態に対して評価):

- **オープンTLD**: 更新は**生成と同じPoW**を要します — 同じチャレンジ(`SCONE-DOMAIN-V1 ‖ canonical name`)、同じラベル長別・ネットワーク別難易度、同じエポック有効ウィンドウ。ドメインが無料で「回収」されることは決してありません: 生存のすべての期間は作業で支払われます。
- **プライベートTLD**: 名前空間はTLD所有者のステーク(PoS)です。トランザクションは**TLD所有者**(またはその委任されたオペレーター)が署名しなければなりません — 署名こそが認可であり、`proof` フィールドは空でなければなりません(20個のゼロバイト)。TLD所有者は、自分の名前空間のドメインを自分のルールで更新します。

### RENEW_TLD (0x8F)

```text
tld(str ≤ 63) tld_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| フィールド | 型 | 制約 |
|---|---|---|
| `tld` | `str` ≤ 63 | 正準TLD名。`tld_id` は `TldId(tld)` と等しくなければならない(再計算) |
| `valid_until` | 可変長整数 | ちょうど `now + 3年` でなければならない(MUST)(固定期間 — 適用時に再導出。その他の値は拒否) |
| `proof` | 有界 | 更新PoW(下記参照) |

TLDの現在の**プライベート**所有者が署名します。適用ルール:

- プライベートTLDにのみ適用(オープンTLDはそのドメインを通じて生きます — 型付き拒否 `TldIsOpen`)。
- 期間中および失効後に続く**60日の猶予**を通じて有効。それを超えればTLDは死んでおり(GC)、拒否は `TldExpiredFrozen`。
- **生成と同等のPoW**: チャレンジ `SCONE-TLD-V1 ‖ name ‖ current_expiry(BE64)`(再利用防止の接尾辞が各PoWを1つの期間に紐付けます)。初期クレームと同じ長さ別難易度。
- 更新は猶予期間中も凍結され**ません** — 期限切れの所有者にまだ残されている唯一の管理操作です。

## 委任権限ファミリー — カウンター紐付け(0xE2、0xA9、0x5E、0x66)

4つのトランザクションが、名前空間に対する権限を委任または再割り当てします。それらは1つのアンチリプレイ規律を共有します: それぞれが、名前空間の**現在の**ミューテーションカウンターについて計算された**カウンターバインディング** — `BLAKE3-256("SCONE-BINDING-MUTATIONS-V1" ‖ id[32] ‖ generation_be64 ‖ counter_be64)` — を運ぶかマイニングします([ミューテーションカウンター](#ミューテーションカウンター)参照)。陳腐化した世代に対して構築されたトランザクションは、その署名やプルーフが依然として暗号学的に有効であっても拒否されます。

### TRANSFER_DOMAIN (0xE2)

```text
domain_id[32] owner[32] new_owner[32] proof.v public_key[32] signature[64]
```

`TransferTld` のドメイン版の鏡像: 現在の所有者が署名し、`new_owner` は不透明。`proof` は、`mutations_binding(domain_id, counter)` — 移転される状態世代のチャレンジ — に対してマイニングされたアンチチャーン譲渡PoW(フラット `transfer_pow_difficulty`、エポックソルト)です。適用ルール: 最初に所有者チェック(`NotOwner`)、次に**現在の**カウンターのバインディングに対するPoW(リプレイされた譲渡は古いチャレンジを解くことになる — 型付き拒否)、その後カウンターをインクリメント。レコードデリゲートセットは**古い**所有者の許可です: 移転はそれをクリアします。`TransferTld` (0x47) は対称なTLD形式です: その譲渡プルーフは `mutations_binding(tld_id, counter)` に対してマイニングされます。

### SET_TLD_OPERATOR (0xA9) / SET_DOMAIN_OPERATOR (0x5E)

```text
id[32] owner[32] operator.option state_binding[32] public_key[32] signature[64]
```

| フィールド | 型 | 制約 |
|---|---|---|
| `id` | 32生バイト | `tld_id`(0xA9)または `domain_id`(0x5E) |
| `owner` | 32生バイト | `public_key` の導出(現在の所有者 / 署名者) |
| `operator.option` | `0x00` または `0x01 ‖ key[32]` | 委任されたオペレーター、または自己運用に戻す `0x00` |
| `state_binding` | **ちょうど32生バイト** | **現在の**カウンターバインディングの正準ハッシュ: `mutations_binding(id, generation, counter)`(generation = TLDなら `valid_until`、ドメインなら `registered_at`)— 陳腐なバインディングは、署名が有効でも型付き拒否(`StaleStateBinding`) |
| `public_key` | 32生バイト | 署名者のEd25519公開鍵(名前空間の所有者) |
| `signature` | **ちょうど64バイト** | 署名対象ペイロードのEd25519署名 |

現在の所有者が署名します(0xA9はプライベートTLD、0x5Eは登録済みドメイン)。アンチレンタル境界: 委任されたオペレーターが務めるシートは最大1つ(`OperatorAlreadyDelegated`)。アプリケーションはバインディングを検証してからカウンターをインクリメントします — **同じ**オペレーターへの再委任も、no-opではなく世代変更です。

### SET_RECORD_DELEGATES (0x66)

```text
domain_id[32] owner[32] count(1) keys[32×count] state_binding[32] public_key[32] signature[64]
```

| フィールド | 型 | 制約 |
|---|---|---|
| `domain_id` | 32生バイト | 対象ドメイン |
| `owner` | 32生バイト | `public_key` の導出 — ドメインの**現在の**所有者 |
| `count` | 1バイト | 最大3(`MAX_RECORD_DELEGATES`) |
| `keys` | 32 × count 生バイト | デリゲート公開鍵、**厳密に昇順**(正準順序。重複や順序外のキーはデコードエラー) |
| `state_binding` | **ちょうど32生バイト** | **現在の**カウンターの正準ハッシュ: `mutations_binding(domain_id, registered_at, mutations)` |
| `public_key` | 32生バイト | 署名者のEd25519公開鍵 — ドメインの**所有者** |
| `signature` | **ちょうど64バイト** | 署名対象ペイロードのEd25519署名 |

KSK/ZSK-lite: 運ばれるセットは前のセットを**完全に置き換えます**(マージなし)。ドメインの所有者とともに、**ちょうどこれらのキー**だけがそのドメインの `UpdateDomain` に署名できます。所有者専用: デリゲートがセットを管理することは決してできません。セットはドメインとともに死に、移転によってクリアされます。アプリケーションは現在のカウンターに対してバインディングを検証してからインクリメントします: `∅ → {9,10} → ∅ → リプレイ` の循環は、取り消されたキーを復活させられません(`security-model.md` のリプレイ攻撃参照)。

## UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

| フィールド | 型 | 制約 |
|---|---|---|
| `domain_id` | 32生バイト | 対象ドメイン |
| `owner` | 32生バイト | `public_key` の導出 |
| `sequence` | 可変長整数 | **> 0**。適用時に `current + 1` と等しくなければならない |
| `record_hash` | 32生バイト | DHTレコードセットへのコミットメント |
| `public_key` | 32生バイト | 署名者のEd25519公開鍵 |
| `signature` | **ちょうど64バイト** | 署名対象ペイロードのEd25519署名 |

注意: `UpdateDomain` は `timestamp` を運び**ません** — 順序付けは `sequence` を通じて行われます。名前も運びません: ドメインはすでに登録されており、IDで十分です。

#### 権限ルール — 所有者またはレコードデリゲート

`owner` フィールドはドメインの**オンチェーン所有者**です(更新はその名前空間に対するものです)。`public_key` は**実際の署名者** — 所有者**または**ドメインのレコードデリゲートキーの1つ(KSK/ZSK-lite、`SetRecordDelegates` 参照)— です。他のすべての署名済みトランザクションと異なり、この型では所有者↔キーの導出は意図的に強制され**ません**(`validate_transaction` がこの型を免除します): 状態層の権限ルール — 署名者 ∈ {所有者} ∪ デリゲート、`sequence == current + 1` — が唯一の束縛です。署名は署名対象ペイロードを通じて `owner` フィールドを依然としてカバーするため、デリゲートは特定の名前空間「のために」署名することになります。注意点: 所有者→キーのマップは初出時に学習される(登録/割り当て)ため、デリゲートによる更新がそのマップを上書きすることは決してありません。

### 厳格な境界

- `public_key`: **ちょうど32生バイト**、長さプレフィックスなし。曲線上の点としてデコードできないpk → デコード時に `InvalidTransactionKey` エラー。
- `signature`: **ちょうど64生バイト**、長さプレフィックスなし、余裕なし。その他のサイズはハードなデコードエラー(切り詰めは拒否)。
- `proof`: 長さ可変長整数 + バイト、256で上限(`MAX_PROOF_LEN`)。アロケーションの前に検査。
- `name`: 長さ可変長整数 + UTF-8バイト、253で上限(`MAX_NAME_LEN`)。`scone-core` の名前付けルールで再検証。

## POW-TO-OWNER — 4つの署名なし形式(0x2A、0x57、0x84、0xC3)

署名付き形式と並ぶ4つの**追加的**変種です(決して置き換えではありません — D1): `REGISTER_TLD_POW`(0x2A)、`RENEW_TLD_POW`(0x57)、`REGISTER_DOMAIN_POW`(0x84)、`RENEW_DOMAIN_POW`(0xC3)。どれも署名者キーも署名も運びません: **マイニングされたプルーフオブワークこそが全認可です**。

### チャレンジは受益者とネットワークを束縛する(D2)

```
challenge = kind_tag ‖ name ‖ pubkey_dest[32] ‖ len(network_id) ‖ network_id
```

- `kind_tag`: ファミリータグ(TLDはSCONE-TLD-V1、ドメインはSCONE-DOMAIN-V1) — クロス種別のリプレイなし;
- `pubkey_dest`: **受益者**のキー — 手渡されたプルーフはそのキーにしか消費されません(プルーフの窃取なし、安全な委託マイニング);
- `network_id`: devnetのプルーフはtestnetやmainnetで決してリプレイされません。

RENEWAL形式は、署名付きファミリーのアンチリユース接尾辞を追加します: `‖ current_valid_until`(BE、8バイト) — 1つのマイニング済みプルーフが正確に1期間を支払います。

### 本体構造

署名付き本体の正確な鏡像で、署名者キーの代わりに `pubkey_dest`、末尾の署名はなし:

- `REGISTER_TLD_POW`: name ‖ tld_id ‖ owner ‖ timestamp.v ‖ proof ‖ open(0x00|0x01) ‖ pubkey_dest — `owner` = hash(pubkey_dest)、再計算、決して信用しない(D4)。open = OPEN誕生(低めのレート)。private = 厳密な3期間、受益者がPoSシートになります;
- `REGISTER_DOMAIN_POW`: name ‖ domain_id ‖ owner ‖ timestamp.v ‖ proof ‖ pubkey_dest — OPEN TLDのみ(プライベート → `TldClosed`: プライベート名前空間は所有者署名パスを保持);
- `RENEW_TLD_POW` / `RENEW_DOMAIN_POW`: name ‖ id ‖ owner ‖ valid_until.v ‖ proof ‖ pubkey_dest。

### 適用ガード(D3)

- オンチェーンの所有者はチャレンジの受益者でも**なければなりません**: renewalは決して持ち主を変えません — 登録だけが新しい財産を作ります;
- 難易度、エポックウィンドウ、猶予、期間は署名付き形式そのものです(D5: 参入コストは変わりません);
- 受益者のキーは、署名者のキーとまったく同じように解決されます(PoSプール解決)。

### txidと共存

txidはコンテンツの正準ハッシュのままです。0x2A/0x57/0x84/0xC3判別子がフォークマーカーです — 古いノードはUnknownDiscriminantとして拒否し、新しいノードはv1形式をそのままデコードします。`signing_payload` は4つすべてに対してN/Aです(型付き `PowNotSignable` エラー): 決して署名されません。

CLI: `scone domain register <name> --to <pubkey-hex>`、`scone domain mine <name> --mine-for <pubkey>`、`scone domain renew <name> --to`、`scone tld register/renew --to`。

## 所有者の導出

厳格なルール: **所有者は常に再計算され、決して信頼されない**。

```text
PublicKeyRef = BLAKE3-256("SCONE-PUBKEY-V1" || public_key)
OwnerId      = BLAKE3-256("SCONE-OWNER-V1" || PublicKeyRef)
```

`owner` が自分の `public_key` の導出と一致しないトランザクションは無効です(コア側では `InvalidOwner`、チェーン側では `OwnerKeyMismatch`)。エンコード、デコード、**そして**チェーン検証で拒否されます(多層防御)。**唯一の例外**: `UpdateDomain` — レコードデリゲートモデルの下では、その `owner` フィールドはドメインのオンチェーン所有者である一方、署名者はレコードデリゲートであり得ます。状態層の権限ルールが署名者を名前空間に束縛します。

## ミューテーションカウンター

すべての名前空間は**単調な**ミューテーションカウンター(`mutations: u64`)を運びます — そのガードされた操作のCASアンチリプレイ世代です。

```text
mutations_binding = BLAKE3-256("SCONE-BINDING-MUTATIONS-V1" ||
                               id[32] || generation_be64 || counter_be64)
```

- **ガードされたトランザクション**(バインディングを検証してからカウンターをインクリメント): `SetRecordDelegates`、`SetTldOperator`、`SetDomainOperator`、`TransferTld`、`TransferDomain`。
- **世代マーカー**: ドメインは `registered_at`(名前空間の全ライフを通じて固定)、TLDは `valid_until`(すべてのライフで構造的に異なる — 再クレームは、完全な複数年期間+猶予の後にしか起こらない)。これがなければ、死んで再登録された名前空間(カウンターリセット)は、カウンターが同じ値に達した時点で、以前のライフのトランザクションを再武装できてしまいます。
- **検証**: アプリケーションは**現在の**状態からバインディングを再導出し、運ばれた値と比較し(セット型)、またはそれに対してPoWを検証します(譲渡)。不一致は、何も変更しない型付き拒否(`StaleStateBinding` / 未解決のプルーフ)です。
- **なぜカウンターなのか**: 現在の状態の純粋関数は、状態が正当に循環して戻ったとき — 譲渡A→B→A、デリゲートセット `∅ → {9,10} → ∅` — に過去の値へ戻ります。リプレイは自分自身を再武装してしまいます。カウンターは決して戻りません。カウンターは名前空間のSMTリーフ(V4)にコミットされ、name-state応答で公開されるため、クライアントは信頼なしにフェッチしてからバインドします(fetch-then-bind)。

## 署名対象ペイロード

署名対象ペイロードは、署名フィールドを除いたトランザクションの正準エンコーディングに、ドメイン分離タグのプレフィックスを付し、**対象チェーンの `network_id` を先頭に注入**したものです:

```text
signing_payload(tx, network) = "SCONE-TX-SIG-V1"
                              || canonical_encode(tx_without_signature, network)
```

ここで `canonical_encode(tx_without_signature, network)` は、上のワイヤー構造に従います(判別子とバージョンバイトを含み、`signature` の手前で停止)。**最初のフィールドとして `network` を伴います** — ネットワークが現れる唯一の場所です。チェーンは自分のネットワークでペイロードを再計算します: 他のネットワーク向けに署名されたトランザクションは署名検証に失敗します(`InvalidSignature`)。これがアンチ・クロスチェーン・リプレイのメカニズムです。

```text
REGISTER_DOMAIN_no_sig = disc(0x21) version(0x01) network name domain_id[32] owner[32]
                            timestamp.v proof public_key[32]
REGISTER_TLD_no_sig    = disc(0x93) version(0x01) network tld tld_id[32] owner[32]
                            timestamp.v proof public_key[32]
UPDATE_DOMAIN_no_sig   = disc(0x52) version(0x01) network domain_id[32] owner[32]
                            sequence.v record_hash[32] public_key[32]
TRANSFER_TLD_no_sig    = disc(0x47) version(0x01) network tld_id[32] owner[32]
                            new_owner[32] proof public_key[32]
REVOKE_TLD_no_sig      = disc(0x6B) version(0x01) network tld_id[32] owner[32]
                            public_key[32]
SET_TLD_OPEN_no_sig    = disc(0xB8) version(0x01) network tld_id[32] owner[32]
                            open(0x00|0x01) open_proof(proof) public_key[32]
ASSIGN_DOMAIN_no_sig   = disc(0xD4) version(0x01) network name domain_id[32] owner[32]
                            assignee[32] proof public_key[32]
RENEW_DOMAIN_no_sig    = disc(0x3C) version(0x01) network name domain_id[32] owner[32]
                            valid_until.v proof public_key[32]
TRANSFER_DOMAIN_no_sig = disc(0xE2) version(0x01) network domain_id[32] owner[32]
                            new_owner[32] proof public_key[32]
```

性質:

- 決定論的かつ純粋: 同じトランザクション ⇒ どこでも同じペイロード。
- ASCIIプレフィックス `SCONE-TX-SIG-V1`(15バイト、16進 `53434f4e452d54582d5349472d5631`)が、プロトコルの他のハッシュドメインでの署名の再利用を防ぎます。
- `signature` 以外のすべてのフィールドがカバーされます — `RegisterDomain` が運ぶ名前を**含む**: どれか1つを変更すれば署名は無効になります。
- 注入された `network_id` はカバーされます: 同じ未署名txを2つのネットワーク向けに署名すると、2つの異なるペイロード(したがって2つの異なる署名)が生成されます。

## 登録プルーフ・オブ・ワーク(PoW)

`RegisterDomain` と `RegisterTld` の `proof` フィールドは、登録プルーフ・オブ・ワークを運びます(`scone-core::pow` モジュール)。

### ダイジェスト(エポックソルト、先行マイニング対策)

```text
epoch  = block_height / POW_WINDOW        (POW_WINDOW = 144 blocks)
salt   = BLAKE3-256("SCONE-POW-V1-ARGON2" || network || challenge || epoch_le64)
digest = Argon2id(m, t, p = 1; password = nonce_le64, salt = salt[0..16])
```

`network` は正準な `network_id` です(M8b): あるネットワーク向けにマイニングされたプルーフは、別のネットワークではプルーフではありません。

`SCONE-POW-V1-ARGON2` は、ネットワークごとのArgon2idパラメータもピン留めします
(`pow_params`: mainnet m = 64 MiB、t = 2、p = 1。testnet/devnet m = 8 KiB、t = 1、p = 1): Argon2idコア(RFC 9106)により、ハッシュあたりのコストはメモリ支配となります(GPU/ASIC耐性)。また、あるパラメータセットの下でマイニングされたプルーフは、別のセットの下で決して検証されません。

`challenge` は、クレームされるオブジェクトの導出入力です — TLDなら `"SCONE-TLD-V1" || tld`、ドメインなら `"SCONE-DOMAIN-V1" || name`。ちょうど `TldId` / `DomainId` のためにハッシュされるバイトであり、プルーフが名前空間横断でもオブジェクト種別横断でも再利用できないようにします。`epoch_le64` と `nonce_le64` は、エポックとノンスのリトルエンディアン8バイトエンコーディングです。

プルーフは、`digest` が少なくとも `difficulty` 個の先行ゼロビットを持つときに有効です。

#### エポックソルト — 先行マイニング対策

ソルトなしのダイジェストは、時間も高さも拘束しません: 誰もが今日、何千もの名前のプルーフを先行マイニングし、mainnetローンチ時に一斉に登録できてしまいます(スクワッティング)— プルーフは自己完結的で、最初に来たブロックならどれでも有効だったからです。ダイジェストは、**トランザクションを適用するブロック**のエポックでソルトされます:

- `epoch = block_height / POW_WINDOW`、`POW_WINDOW = 144`ブロック
  (65秒/ブロックで約2.6時間): 正直なユーザーのための猶予ウィンドウ(マイニングし、ウィンドウ内に提出)。それを超えれば再マイニングが必須。
- プルーフはワイヤー上で自分のエポックを申告します。適用時、申告されたエポックは、適用ブロックの**有効ウィンドウ**に入らなければなりません — 後方のみの許容(遅い提出は吸収され、「未来のための」マイニングは不可能)。ウィンドウの大きさは `epoch_validity_window(difficulty)` = `1 << max(0, difficulty − 16)` エポック(拡張ウィンドウ: mainnetの高難易度は数日〜数週間かかり、1エポックより長い。ウィンドウ ≈ マイニングコストのため、備蓄は何の得にもならない — 一斉登録の各プルーフは、ウィンドウ内でその完全な `2^difficulty` ハッシュのコストがかかる)。難易度 ≤ 16ビット(testnet、長い名前)では、厳密等値ルールは変更なし。
- メモリプール准入(ベストエフォート): 未来のブロック高は未知 — リレーは `現在の高さ + 1` のエポックに対して検証します。適用時の厳密な検証が権威であり続けます。
- 決定論的リプレイ: プルーフは自分のエポックを運び、ウィンドウは(難易度、ブロックエポック)の純粋関数です — 履歴ブロックは、どのノードでも同一に再検証されます。

mainnet前の破壊的変更: 古いコンパクト形式(12バイト)は、後方互換性なしに拒否されます。

### ワイヤーペイロード(20固定バイト)

```text
nonce_le64 || epoch_le64 || difficulty_le32
```

難易度とエポックはプルーフと共に運ばれますが、それらは**生成者の申告であり、したがって決して信頼されません**: 検証者は、難易度を登録種別の**プロトコル定数**と比較し(申告された難易度が定数と異なるペイロード — 低くても*高くても* — は拒否、`InvalidProof`)、エポックを適用ブロックのものと比較します(不一致なら `ProofEpochMismatch`)。続いてノンスが定数難易度で再検証され、ダイジェストはゼロから再計算されます。

### 難易度(**長さベース**曲線、M8b + 実測較正)

難易度はフラットなパラメータではありません: ネットワークごとに、**生成されるラベルの長さ**の決定論的関数です(`scone-core::pow_length` — `RegisterTld` ならTLD、`RegisterDomain` なら左端ラベル):

```text
difficulty(len) = apex − round(0.415 × (len − 1)), clamped at the floor
```

| ネットワーク | プライベートTLD(頂点/下限) | 生まれながらオープンのTLD / ドメイン(頂点/下限) |
|---|---|---|
| testnet | 8ビット / 2ビット | 6ビット / 2ビット |
| mainnet | 42ビット / 30ビット | 40ビット / 30ビット |
| devnet | 2ビット / 1ビット | 1ビット / 1ビット |

**生まれながらオープン**なTLD(`RegisterTld` で `open = 0x01`)は「オープン化」曲線を支払います(頂点 − 2、`domain_difficulty` と同様)。**既存のプライベートTLDのオープン化**(`SetTldOpen`)はフラット難易度のままです
(`open_tld_pow_difficulty`: testnet 6ビット / mainnet 22ビット /
devnet 2ビット — txは `tld_id` のみを運び、名前は生成時にその長さの対価を支払済み)。
Mainnetベンチマーク(プライベートTLD、約6 Mハッシュ/秒): 1文字 = 42ビット(≥ 7日)、5文字 = 40ビット(約2日)、19文字 = 35ビット(約1.6時間)、29文字以上 = 30ビット下限(約3分)。曲線は `scone-core::pow_length` にあります(定数 `MAINNET_APEX = 42`、`MAINNET_FLOOR = 30`、`TESTNET_APEX = 8`、`TESTNET_FLOOR = 2`、`DEVNET_APEX = 2`、`DEVNET_FLOOR = 1`)。フラットなパラメータ(`tld_pow_difficulty` など)は `scone-core::network::NetworkParams` に留まります。値の変更 = **そのネットワーク**のコンセンサス変更です。不変条件: devnetは完全な第3の `NetworkParams` であり、testnetはdevnet前の定数と厳密に同一のまま(Argon2idパラメータ、曲線、ウィンドウ)— devnetは、コード自体を超えて、他のネットワークと定数を共有しません。

フラットな頂点の実測較正(2026年10月、`cargo run --release -p scone-core --example pow_bench` — 参照x86-64でシングルスループット約7.7 M H/s):

| 種別(mainnet) | 難易度 | 平均実時間 |
|---|---|---|
| `RegisterTld` | 24ビット | 約7秒 |
| `RegisterDomain` | 20ビット | 約54 ms |

約65秒/ブロックで: TLDクレームはシングルコアCPUの約0.1ブロック、ドメインはブロックの約1 ms — ユーザーには象徴的、大規模な産業化には高価(アンチスクワット)。ハードウェアの目標が低下したら再較正すること(モバイルARM: 約4〜8倍遅い)。

状態分岐(M8b、実装済み): `RegisterTld` は常にPoWを要求。`RegisterDomain` は**オープン**TLD上で要求(クローズパスはassign専用 — `AssignDomain`、TLD所有者が署名し、独自のフラットなアンチスパムプルーフを持ちます)。

**譲渡とアサインのプルーフ(コンセンサスレベルのアンチスパム)**: `TransferTld`、`TransferDomain`、`AssignDomain` はそれぞれ**フラット**なプルーフオブワークを運びます — 2つの譲渡には `transfer_pow_difficulty`(チャレンジ = 名前空間の**現在の**カウンターバインディング、[ミューテーションカウンター](#ミューテーションカウンター) — フラットな `SCONE-*-XFER-V1` タグは廃止されました)、アサインには `assign_pow_difficulty`(チャレンジ `SCONE-DOMAIN-ASSIGN-V1 ‖ name`)。登録と同じエポックソルトとアンチプレマイニングウィンドウ。メモリプール准入時に検証され(推定エポック、ベストエフォート)、適用時に厳密に検証されます。目標: 譲渡の頻繁な売買やプライベートTLD下での大量作成のコストは、ポリシーではなくコンセンサスレベルの仕事にします。長さごとの課税なし: アサインについては、名前空間の排他性はプライベートTLDの作成時にすでに支払われています。

**受動的な譲渡**: これらの取得パス(`TransferTld`、`TransferDomain`、`AssignDomain`)は譲渡者だけが署名します — 受取人は決して相談されません。意図された選択です(贈与、デッドドロップ、遺贈)。課せ可能なPoS債務はありません: `security-model.md`(受動的取得と適格性)を参照。

## 検証ルール

トランザクションが有効なのは、次の場合のみです:

1. **厳格なデコード**: 上記の構造と境界が尊重されていること(既知の判別子、バージョン `0x01`、pk 32 B、署名64 B、デコード時にコアの不変条件を再検査)。
2. **所有者/pk結合**: `owner == OwnerId::from(public_key)`(再計算)。
3. **名前/ID整合**(RegisterDomain): `domain_id == DomainId(name)`(再計算)。
4. **署名**: チェーンのネットワークで**ゼロから再計算された**署名対象ペイロードに対する、`signature` のEd25519 `verify_strict`(ピアからも、提供されたトランザクションからも決して取らない)— 他のネットワーク向けに署名されたtxはここで失敗します(`InvalidSignature`、アンチ・クロスチェーン・リプレイ)。
5. **状態ルール**(ブロック内での適用時 — M8b、`/docs/en/technical/blockchain.md` 参照):
   RegisterTld → TLD空き + ICANNでない + PoW(長さベース難易度)。RegisterDomain → TLD登録済み**かつオープン** + ドメイン空き(猶予を含む)+ PoW(長さベース)。
   UpdateDomain → ドメイン存在、現在の `owner`、`sequence == current + 1`。
   TransferTld / RevokeTld / SetTldOpen / AssignDomain → 存在**かつプライベート**なTLD、署名者 = TLDの所有者。
   TransferDomain → ドメイン存在、所有者。
   RenewDomain → ドメイン存在、厳密な延長 ≤ 3年、
   PoW認可(オープンTLD: 生成と同じPoW、**現在の失効に紐付け** — 登録および過去すべての更新と異なるチャレンジ)またはPoS
   (プライベートTLD: TLD所有者 — またはその実効オペレーター — の署名、空の `proof`)。

検証は、信頼できないデータに要求されるるとおり、`verify_strict` を使います(なよなよした署名と、弱い/非正準なキーの拒否)。

## `push_block` での操作順序

暗号学的検証(2〜4)は、状態適用の**前に**、各トランザクションに対して、ブロックごとのアトミックな作業状態上で行われます。いずれかのエラーがあれば、チェーンは厳密に変更されないままです。

## ブロック組み立て

`BlockBuilder`(`scone-blockchain`)は、トランザクションキューからブロックを組み立てます: 順序付きリストに対して `tx_root` を再計算、ヘッダーは親ハッシュにチェーン(`prev_hash`)、`version` = `PROTOCOL_VERSION`。コンセンサスペイロード(署名済み生成者ペイロード)とタイムスタンプは呼び出し側が提供します。既存の `Consensus` トレイトが、検証と対称に使われます。

## 16進例(実際の再現可能ベクトル — M8b)

### RegisterDomain `example.uip`、タイムスタンプ42、testnet

署名対象ペイロード(ピン留めベクトル。歴史的には削除されたオフラインツールで生成 — 値は有効なベクトルのまま)。ここに `network` が現れるのは、これが署名対象ペイロード(その唯一の場所)だからです。署名済みワイヤートランザクション自体には含まれません:

```text
53434f4e452d54582d5349472d5631       SCONE-TX-SIG-V1 prefix
21                                    disc REGISTER_DOMAIN (0x21)
01                                    format version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
0b 6578616d706c652e756970             name (str, length 11)
e6a2cbe264f81ed7da4d901c4deec6a3579224b8080fad01cb7461b8c264f181  domain_id
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner (fixed build key)
2a                                    timestamp (varint, 42)
14 0200000000000000 0000000000000000 04000000
                                      proof (20 B: nonce 2, epoch 0, difficulty 4 — testnet)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk + start of sig
```

### RegisterTld `uip`、タイムスタンプ42

署名対象ペイロード(ピン留めベクトル)— `TldId` は `/docs/en/general/naming.md` でピン留めされた導出ベクトルです:

```text
53434f4e452d54582d5349472d5631       SCONE-TX-SIG-V1 prefix
93 01                                 disc REGISTER_TLD + version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
03 756970                             tld (str, "uip")
ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e  tld_id ("uip")
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner
2a                                    timestamp (varint, 42)
14 8e00000000000000 0000000000000000 07000000
                                      proof (20 B: nonce 0x8e, epoch 0, difficulty 7 — testnet, 3-char label)
00                                    open (0x00 = private birth)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk
```

(完全な値は、削除されたオフラインツールで再現可能でした。下のベクトルはピン留めされ有効なままです。PoWはネットワーク難易度で自動的にマイニングされます — ノンスは実行ごとに変わります。安定なのは network/name/id/owner/pk だけです。2026年9月のワイヤー変更以降、署名済みトランザクションの16進には、バイト `0d 73636f6e652d746573746e6574` はもはや含まれません: ネットワークは、署名対象ペイロード内でのみ転送されます。)

## 16進が今いる場所

オフラインの `tx build|sign|verify` コマンドは削除されました
(cli-simplify-v1)。署名済み16進フォーマット自体は変更なく、**web/APIフロー**のワイヤーフォーマットのままです: レジストラWebアプリは、正準16進としてAPI越しに、トランザクションを構築・署名し(ブラウザ内keystore)、提出します。CLI側では、ユーザーパスはワンコマンド形式です:

```bash
scone domain register example.uip        # mines, signs, submits
scone domain renew example.uip                      # renewal proof, signs, submits
scone domains                            # what you own
```

すべてのコードパスで、所有者とキーは署名識別情報から**再計算**され、いかなる入力からも取られません。

## バージョン履歴

| バージョン | 内容 | 状態 |
|---|---|---|
| 1 | 埋め込み `public_key` + `signature`、ペイロード `SCONE-TX-SIG-V1`、バージョンバイト `0x01`、`RegisterDomain` が名前を運ぶ、`RegisterTld` (0x93)、M8aファミリー: `TransferTld` (0x47)、`RevokeTld` (0x6B)、`SetTldOpen` (0xB8)、`AssignDomain` (0xD4)、`RenewDomain` (0x3C) — 規範ワイヤーフォーマット。M8b: `network` フィールドがすべてのペイロードの先頭で署名される、`RegisterTld` が名前を運ぶ、完全な状態ルール、ネットワークごとのPoW難易度 | 廃止 |
| 1 (2026-09、networkなしのワイヤー) | **すべてのトランザクションのワイヤーからの `network` フィールドの除去**: network_idは署名ペイロード内にのみ存在(`signing_payload(tx, network)` が先頭に注入)。ネットワーク分離は署名に依存(よそのペイロードには `InvalidSignature`、`WrongNetwork` エラーは消滅)。`TX_FORMAT_VERSION` は `0x01` のまま(再起動可能なtestnet、共存なし)。**すべてのTxIdが変わる**(TxIdは署名済みワイヤーエンコーディングのハッシュであり、14〜15バイト失う)。PoWダイジェスト、ジェネシス、ネットワークごとのデータディレクトリは変更なし | 廃止 |
| 1 (2026-10、PoWエポックソルト) | **先行マイニング対策PoWダイジェスト**: ソルト `BLAKE3("SCONE-POW-V1-ARGON2" ‖ network ‖ challenge ‖ epoch_le)` + `nonce_le` 上のArgon2idコア。`epoch = block_height / 144` はtxを適用するブロックのもの。プルーフのワイヤーペイロードは12 → **20バイト**(`nonce_le ‖ epoch_le ‖ difficulty_le`)。申告されたエポックは適用ブロックのエポック有効ウィンドウに対して検証(型付き拒否 `ProofEpochMismatch`)。メモリプール准入ではベストエフォートの `(height+1)/144`、オフラインパスに `--epoch` CLIフラグ。mainnet前の破壊的変更: 古い12バイトペイロードは拒否、**すべてのTxIdが変わる**(`proof` フィールドが8バイト増える) | 廃止 |
| 1 (2026-10、ミューテーションカウンター) | **CASアンチリプレイバインディング**: 名前空間ごとの単調なミューテーションカウンター(`DomainState`/`TldState::Private`)。5つのガードされたトランザクションが検証してからインクリメント(`SetRecordDelegates` 0x66 **新規**、`SetTldOperator`、`SetDomainOperator`、`TransferTld`、`TransferDomain`)。譲渡PoWはカウンターバインディングに対してマイニング(静的な `SCONE-TLD-XFER-V1`/`SCONE-DOMAIN-XFER-V1` チャレンジは廃止)。セット型は署名済み32バイトの `state_binding` ワイヤーフィールドを運ぶ。SMTリーフV4がカウンターをコミット(V2/V3は廃止 — V3は出荷されなかった)。ストレージコーデックはドメインあたり+8バイト / プライベートTLDあたり+8バイト。`NameState` がカウンターと `registered_at` を公開。mainnet前の破壊的変更: 状態フォーマット+ワイヤー変更、**testnetリセット**、古いスナップショットはインポートで拒否 | 現行 |
