# RFC — 運用プルーフとファイナリティ委員会(proof-of-service)

> 状態: 規範(NORMATIVE)— エンドツーエンドで実装済み(ステップ1〜5:
> core/protocol/blockchain/network/CLI。本文書は統合形です)。以下のすべてのルールはコードに対して検証済みであり、引用される定数はコード上の値です。
> 日付: 2026年10月(規範統合)。関連: `scone-core`、`scone-protocol`、`scone-blockchain`、`scone-network`、`scone`(CLI)。

## 1. 動機

ファイナリティプールは所有者プールです(生存ドメイン/TLDの所有者)。
所有はシートを与えますが、**運用**は何も与えません。このRFCは、所有権を条件付けることなく、チェックポイント署名の権利をオンラインリレーの運用プルーフに結び付けます:

- **保護される自由**: プライベートTLDは、期間が更新され続ける限り(`RenewTld`、3年間の期間)、活動条件なしで割り当て可能なままです。*チェックポイント署名*だけが、存在によって獲得されます。
- **運用へのインセンティブ**: プライベートTLD(または任意のシート)の運用 = 委員会に着席すること。オペレーターはネットワークを稼働させ続けることに参加し、プロトコルはそれを認めます。
- **測定可能な分散化**: 委員会は、休眠したキーではなく、*実際にオンラインの*オペレーターの集合になります。

## 2. 概要

軽量な `Heartbeat` トランザクション(判別子 `0x71`)。**オペレーター**キーによって署名され、エポックごとに1回発行されます。状態は直近のハートビートのスライディングウィンドウを保持します。エポックの委員会は、**実効オペレーターが新鮮なハートビートを持つプールの部分集合**から選出されます。ハートビートがなければ:署名はできませんが、所有権の権利は失われません。

## 3. Heartbeatトランザクション

```
Heartbeat {
    operator: PublicKey,     // Ed25519 key of the operator (the signer)
    owner: OwnerId,          // identity RE-DERIVED from operator (never trusted)
    tip_seen: [u8; 32],      // last tip seen by the node
    height_seen: u64,        // corresponding height
    timestamp: u64,          // Unix seconds
} + Ed25519 signature over the canonical payload (network injected)
```

署名対象本体(ワイヤー、`scone-protocol`):
`operator[32] ‖ owner[32] ‖ tip_seen[32] ‖ height_seen.v ‖ timestamp.v ‖ operator[32]`(署名者のキーは、あらゆるtxと同様に本体の末尾に現れます)`‖ signature[64]`。エンコードでおよそ230バイト — **PoWなし**:コストは継続的な存在です(エポックごとに1回のハートビートを、何ヶ月も)。

署名者はオペレーター自身です: `owner` は常に `operator` から再導出されます(入力から取ることは決してなく)、デコードは、先頭の `operator` フィールドと異なる署名キーを型付きエラー(`InvalidOwner`)で拒否します。

- `tip_seen`/`height_seen` は観測性に役立ちます(バグのある、または遅れたオペレーターの検出)。有効性には影響しません(わずかに遅れたノードも、依然として生きているノードです)。

### 3.1 純粋検証(scone-core)

1. ネットワークの正準ペイロードに対する有効なEd25519署名。
2. `owner` == `operator` から導出された識別情報(そうでなければ `InvalidOwner`)— エンコード時、デコード時、そして `validate()` によって検査。

### 3.2 状態ルール(適用、scone-blockchain)

高さ `block_height` の運搬ブロック内のハートビートの適用時(すべての拒否はあらゆる変異に先行):

1. **エポック横断アンチリプレイ**: オペレーターごとに `timestamp` は厳密に増加 — `timestamp <= operator_last_hb[operator]` なら型付き拒否 `StaleHeartbeat`。
2. **エポックごとの一意性**: (`operator`、運搬ブロックのエポック)ごとに1つのハートビートのみ受理 — 重複は型付きエラーで拒否(`HeartbeatDuplicate`)。エポックは `block_height / epoch_min_blocks`(ファイナリティと同じ定義。新しい定数は決して導入しない)。
3. ウィンドウへのジャーナル記録付き挿入 + `operator_last_hb` の更新(リオルグ時に可逆、ビット完全なアンドゥ)。

### 3.3 アンチスパム

- オペレーターごと、エポックごとに1つの受理tx — 最大フローは `pool_size / epoch_secs` で、無視できるほどです。
- 手数料なし(経済圏の前):レートは構造的に有界です。

## 4. 運用委任

シート → オペレーターのリンクは、2つの専用トランザクションを通じて行われます:

- `SetTldOperator { tld_id, operator }`(判別子 `0xA9`)。**プライベート**TLDの**現在の**所有者が署名します(オープンTLDには正規に署名する相手がいません — 型付き拒否 `TldIsOpen`)。`operator: None` = 取り消し。所有者のキーによる自己運用に戻ります。即時に取り消し可能。
- `SetDomainOperator { domain_id, operator }`(判別子 `0x5E`)。ドメインの現在の所有者が署名します。同じ意味論。
- `RegisterTld` / `RegisterDomain` / `AssignDomain` は `operator` フィールドを運びません:シート → オペレーターのリンクは常に、チップで解決される、最近のオンチェーンの事実です。

ワイヤー上のオペーターオプションは正準です: `0x00` = `None`(自己運用)、`0x01` = 生の32バイトキー。その他の値はデコード時に拒否されます。

状態は次を維持します:

```
seat_operators: HashMap<OwnerId, OperatorKey>   // per seat
operator_seats: HashMap<OperatorKey, u64>       // bounded counter
operator_last_hb: HashMap<OperatorKey, u64>     // last accepted ts
```

適用ルール(`apply_seat_operator`、ジャーナル記録付き):

- **冪等性**: 同じオペレーターへの再委任(または存在しない委任の取り消し)は、ジャーナル記録なしのno-opです(ビット完全なアンドゥがそれを要求します)。
- **レンタル対策の上限**: オペレーターは、同時に**ただ一つのシート**からのみ委任され得ます(`MAX_SEATS_PER_OPERATOR = 1`)。すでに取られたオペレーターに2番目のシートが委任しようとすると → 型付き拒否 `OperatorAlreadyDelegated`。理由: 上限がなければ、大規模オペレーターが署名を大量に売り、中央集権を再現します。それはまさに、選出委員会が避けるべきものです。

## 5. 新鮮さのウィンドウ

### 5.1 スライディングウィンドウ

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- 有効なハートビートの適用時にエントリ挿入(§3.2)。
- すべてのブロック適用での決定論的パージ(`gc_heartbeats_journaled`): エポック `e` のエントリで `e + HEARTBEAT_WINDOW_EPOCHS <= epoch` を満たすものは除去(ジャーナル記録付き — 失効キューと同様にリオルグで可逆)。
- `HEARTBEAT_WINDOW_EPOCHS = 2`: 現在のエポック + 前のエポック — オペレーターは1つのエポックを逃しても新鮮さを失わず、2つ目で失効します。
- **構造的なメモリ上限**: ウィンドウは最大 `HEARTBEAT_WINDOW_EPOCHS × operator_count` エントリ(オペレーターごと、エポックごとに1つ、タプルキー)。`operator_last_hb` はオペレーター数で有界であり、パージされません(アンチリプレイルールは単調です)。

### 5.2 新鮮さ

「新鮮」= `e` が `[epoch − HEARTBEAT_WINDOW_EPOCHS + 1, epoch]` にあるエントリ `(e, operator)` の存在。状態はすべてのブロックでパージするため、`recent_heartbeats` 内の存在がすなわち新鮮さです(`operator_is_fresh`)。

### 5.3 実効オペレーター

シートの**実効オペレーター**は、生存する委任が存在すれば `seat_operators[seat]`、そうでなければ所有者のキーです(`effective_operator`)。シートはその実効キーの下で署名します — 委任は署名権を移動させるのであって、所有権を決して移動させません。

## 6. 委員会選出

```
eligible = { effective operators of the frozen pool's seats
             (banned key excluded) having a fresh heartbeat }

if |eligible| >= MIN_FINALITY_COMMITTEE_SIZE (= 4) :
    committee = select_committee(seed, eligible)   // existing draw
else :
    committee = select_committee(seed, pool)       // safe degradation
```

規範的詳細(`finality.rs` で検証、`/docs/en/technical/blockchain.md` §「Seats and the finality committee」で仕様化):

- `pool` は、**最後にファイナライズされたチェックポイントで凍結された**プールです(`FinalizedBase.pool` — 決してライブ状態ではない): ローカルビューはノード間で分岐し、クォーラムは決して形成されません。
- `seed` は前のチェックポイントの内容ハッシュ(`signing_hash`)から導出されます。
- 凍結プールのすべてのシートは、チップにおける状態によって実効オペレーターに解決されます(委任とハートビートは最近のオンチェーンの事実: 同じチップにいる2つのノードは同じ集合を計算します)。追放されたキーは除外されたままです。
- 抽選は、ソート・重複除去された実効キーの集合に対して実行されます(`select_committee` 変更なし)。

**明示的で安全な縮退**: ブートストラッピング中のネットワーク(ハートビートなし)、またはオペレーターの全面的な停止は、プール全体にフォールバックします — 構成上、ファイナリティは決して失われません。これはこの場所に適用された、プロジェクトの「決してデッドロックしない」ルールです。

### 6.1 コールドブートストラップ

エポック0(および `MIN_FINALITY_COMMITTEE_SIZE` 未満のオペレーターしか新鮮でない間): プール全体。オペレーターが発信するにつれて、制約は自然に引き締まります。設定すべき特別な移行期間はありません。

## 7. 自動発信(scone-network)

すべての生成サイクルで(`Wake::Produce` — 新しいタイマーなし)、リレーはエポック境界を跨いだかどうかを確認します(`height / epoch_min_blocks`、ファイナリティと同じ定義)。新しいエポックの最初のサイクルで、次の条件を満たす場合にハートビートを発信します:

1. アンカーキーが武装されている(`--anchor-key` または自動)、かつ
2. そのキー自体が生存するシートを保持している(ローカル検査: キーがオンチェーンの適格プール内 — 生存ドメイン/TLDの所有者、追放キーは除外)。

ローカルアンチスパム: ノードごと、エポックごとに最大1回の発信(`last_emitted_epoch` メモ、ローカルメモリ、決してチェーン状態ではない)。メモは、メモリプールへの注入が成功した後にのみ取得されます — 失敗すれば次のサイクルで再試行です。ここには致命的なものは何もありません。最悪でも、ハートビートは次のサイクルを待ちます。

**委任されたオペレーター**(自身は所有者でない)は、自動発信の対象ではありません: §8のCLIビルダーを通じて発信します(手動またはエアギャップ運用)。ハートビートは、発信者自身によってブロックに生成されるとは限りません: それは他のあらゆるtxと同様です(メモリプール、誰による生成でも)。

## 8. CLI(`scone`)

オフラインハートビートビルダーは、`tx` サーフェスとともに削除されました(cli-simplify-v1でオフラインtxサーフェスと共に)。リレーは自分のハートビートを自動的に発信します(§7)。手動/エアギャップ発信が再び必要になれば、専用コマンドとして戻ってくるでしょう。デフォルトは次でした: `tip_seen` = 32個のゼロバイト(観測性のみ)、`height_seen` = 0、`timestamp` = now。
- あらゆるtxと同様に、`tx sign` はオペレーターと所有者を実際の署名キーに**再結び付け**ます(入力値は決して信頼しない)。`tx verify` はローカルで検証します。

## 9. 観測性

- `NodeStatus`(ワイヤーp2pメッセージ `0x0d`)は `fresh_operators: u32`(可変長整数)を運びます: 新鮮なハートビートを持つオペレーターの数 — 委員会の適格となるプールの生存部分集合(プールのシートの実効オペレーター、追放キー除外、ソート・重複除去)。リレーが `state().fresh_operators(height)` から充填し、`scone status` が `fresh operators: N` として表示します。
- キャンペーンの論拠: 「プライベートTLDの運用 = 分散型DNSのファイナリティ委員会に着席すること」— オンチェーンで検証可能。
- 将来のサーフェス(未確定): `operator_status` — シートごとに、委任されたオペレーター、新鮮さ(新鮮 / 失効 / 未出現)、最後の `tip_seen`。ローカル制御RPCは削除されました(p2pのみのノード)。サーフェスはまだ定義されるべきです。

## 10. このRFCがしないこと

- 非活動による**所有の喪失なし**(オプションA — 運用されないTLDの減衰 — は明示的に拒否済み)。
- **報酬なし**(オプションCは経済層を待つ)。
- ハートビートはサービスの*質*(生成されたブロック、提供されたDHTレコード)を証明しません — 存在だけです。実際の貢献の計数は、よりゲーム可能な将来の拡張であり得ます。

## 11. 保護される不変条件

- ビット完全な決定論: ウィンドウと選出は、適用された状態の純粋関数です(ジャーナル記録、ビット完全なアンドゥ)。
- ネットワークデータで決してパニックしない: 不正なハートビートや委任 → 型付き拒否(`InvalidOwner`、`StaleHeartbeat`、`HeartbeatDuplicate`、`OperatorAlreadyDelegated`、`TldIsOpen`、`NotTldOwner`、`NotOwner`)。
- 所有は無条件。条件付きなのはチェックポイント署名だけです。

## 12. 定数とパラメータ(コード上の値)

| 名前 | 値 | 場所 |
|---|---|---|
| `HEARTBEAT_WINDOW_EPOCHS` | 2 | `scone-blockchain/src/state.rs` |
| `MAX_SEATS_PER_OPERATOR` | 1 | `scone-blockchain/src/state.rs` |
| `MIN_FINALITY_COMMITTEE_SIZE` | 4 | `scone-core/src/consensus_params.rs` |
| `Heartbeat` 判別子 | `0x71` | `scone-protocol/src/transaction.rs` |
| `SetTldOperator` 判別子 | `0xA9` | 同上 |
| `SetDomainOperator` 判別子 | `0x5E` | 同上 |

これらの値はコンセンサスの一部です。変更すると、チェーンのリプレイ可能性(ウィンドウ/パージ)や選出(委員会)を壊します。

## 13. 実装計画 — 最終状態

| ステップ | クレート | 内容 | 状態 |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`、`SetTldOperator`、`SetDomainOperator`、ワイヤー、検証 | 完了 |
| 2 | blockchain | スライディングウィンドウ(ジャーナル記録付き)、フィルタ済みプール、選出、約15テスト | 完了 |
| 3 | network | 自動発信、txリレー | 完了 |
| 4 | CLI | `tx build heartbeat`、`scone status` の `fresh operators` 表示 | 完了 |
| 5 | docs | このRFCの規範形への統合、fr/enミラー | 完了 |
