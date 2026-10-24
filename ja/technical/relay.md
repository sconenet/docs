# ネットワークリレー(M4)— アーキテクチャ、フロー、CLI制御

> Sconeネットワークリレー(`scone-network`)の規範仕様。
> ドキュメントとコードの不一致はすべてバグです(信頼できる正しい情報源を修正すること)。

## 概要

**リレー**はSconeのフルP2Pノードです: 正準ブロックチェーン(RAM + `NodeStore` redb)を維持し、トランザクションとブロックをリレーし、ブロックを生成し(devnetモード)、同期プロトコルを提供し、Kademlia DHT内で `SignedDnsRecord` を公開・解決し、CLIが直接問い合わせる**p2pプロトコル**(`/scone/reqres/1`)を公開します(RPC→p2p統合: JSON TCPチャネルは消滅)。

```text
                    ┌────────────────────────────────────────┐
                    │              scone relay               │
  P2P (libp2p)      │                                        │
  QUIC (udp,        │  Swarm libp2p                          │
  ephemeral port    │  ├─ identify  (address discovery)      │
  by default) ◄─────┤  ├─ ping      (liveness)               │
                    │  ├─ kad       (record DHT)             │
                    │  └─ reqres    (scone-prot. messages)   │
                    │                                        │
                    │  Relay core (single task)              │
                    │  ├─ Blockchain (full validation)       │
                    │  ├─ Mempool (bounded, dedup by TxId)   │
                    │  ├─ BlockBuilder (devnet production)   │
                    │  ├─ Fork parking (bounded, 2048)       │
                    │  └─ Sync driver (batched catch-up)     │
                    │                                        │
  CLI ◄─────────────┤  reqres p2p (QUIC, same messages)      │
                    └────────────────┬───────────────────────┘
                                     │
                              NodeStore (redb)
```

`crates/scone-network` はワークスペースで**唯一のasyncクレート**です: その下のすべて(crypto → core → protocol → blockchain → storage)は同期的かつ純粋なままです。

## 接続准入(P1.5)

スワームレベルでのハードリミット(libp2p `connection_limits` 動作、`SconeBehaviour` の最初のエントリ):

- `MAX_TOTAL_CONNECTIONS = 128` — リレーごとの確立済み接続(着信 + 発信)の上限。超過する着信はREFUSED。キューイングは決してなし。
- `MAX_PER_PEER_CONNECTIONS = 4` — 単一のピアが、多数の接続を開いて総クォータを独占できないようにします。
- `MAX_PENDING_INCOMING = 32` — slowlorisガード: 未認証のハーフ接続が無期限に待機することは決してありません。

これらのリミットは、既存のメッセージごとのリミット(`MAX_MESSAGE_LEN` 1 MiB)とバジェットごとのリミット(同期 `max ≤ 128` ブロック、DNS TCP 64接続/512 B/5秒)を補完します。接続洪水攻撃者が得るのは拒否であって、メモリの増大ではありません。

## ピア発見

3つの相補的メカニズムが、この優先順位で働きます:

1. **明示的シード**(`--bootstrap`、完全なmultiaddr)— 常に最優先。空でないリストの提供は他のすべてを置き換えます。`--no-builtin-seeds` は、100%プライベートなデプロイのために組み込みシードを空にします。
2. **組み込みシード**(`scone-network/src/config.rs` の `builtin_seeds(network)`)— 各公式バイナリリリースで凍結、ネットワークごと。未設定のリレーは単独でネットワークに参加します。(公開testnetがデプロイされるまで、testnetでは空。)
3. **学習済みディレクトリ**(`<data-dir>/peers.json`)— `identify` で学習したすべてのアドレスを保持(最終_seen。上限**2000エントリ** — 接続ではなくアドレスで、最大時でもRAM約400 KiB — そして**90日の年齢切断**: 90日以上前に最後に見られたエントリは、書き込み時に破棄され、ロード時に無視されます。2000スロットを、起動時のダイヤルタイムアウト以外の貢献しかない回転前の化石ではなく、もっともらしいピアで満たし続けます)。保存は生成ティックで約1回/分。シードもブートストラップもない起動時、ディレクトリが再構成されます: リレーは設定なしでいつものピアを見つけます。これはキャッシュです: ファイルの欠落・破損 = 空のディレクトリ。決してエラーではありません。
4. **UPnP**(`--upnp`、オプトイン)— ホームルーターによるQUICリッスンポートのマッピング: ノードは、サーキットリレーなしに外から到達可能になります。`NewExternalAddr` で、外部アドレスがスワームの実際の外部アドレスとして登録されます(`add_external_address` — identify/autonatに見え、トランスポートリレーの候補)。IGDゲートウェイなし(`GatewayNotFound`)または非ルーティング可能なゲートウェイ(`NonRoutableGateway`): 通常の情報ログのみで、マッピングはスキップ — ホームLANでは正常なケースです。期限切れのマッピング(更新失敗)は警告です。

UPnPは**デフォルトで無効**のままです(`config.rs` の `enable_upnp` = `false`): 公開デプロイは同意なしにどのルーターとも話しません。**mDNS発見は削除されました**
(remove-mdns-discovery、2026-10-22): ピアリングは、明示的なブートストラップ、組み込みシード、Kademlia のみから来ます — レガシーな `--mdns`/`SCONE_MDNS` フラグは効果なしで引き続き受け付けられます(スクリプト互換性、`--rpc` と同じ扱い)。

### ローカルトポロジー最適化(常時有効)

リレーは、純粋な `scone-topology` エンジンを使って能動的に近傍を最適化します(クレートのドキュメント参照)。**常に有効**。公開されている調整は**プロファイル**だけです(`--topology-profile`、環境変数 `SCONE_TOPOLOGY_PROFILE`、デフォルト `default`):

| プロファイル | min / target / max | 評価 | 誰向けか |
|---|---|---|---|
| `tiny` | 2 / 4 / 6 | 120秒 | Raspberry Pi、ホームボックス、1-vCPU VPS |
| `small` | 4 / 8 / 12 | 90秒 | 小型サーバー、ホームNAS |
| `default` | 8 / 16 / 32 | 60秒 | 標準サーバー(デフォルト) |
| `high` | 16 / 32 / 64 | 45秒 | 主権ノード、バックボーン |
| `very-high` | 32 / 64 / 128 | 30秒 | 創設者ハブ |
| `devtest` | 2 / 4 / 6 | 5秒 | **ラボ/テスト専用**(設計上不安定) |

```
scone relay --topology-profile tiny
```

保護パラメータ(ヒステリシス0.15、クールダウン600秒、猶予120秒、測定15秒)はどこでも同一です: それらは安定性を守り、ノードサイズに応じてスケールする理由はありません。未知の名前は型付き起動エラーです。

不整合な設定(`min > target > max`、`[0,1]` の外のしきい値 — `NaN` を含む —、ゼロ間隔)は、**型付き起動エラー**です。決して黙った動作ではありません。

**ビュー(`TopologyView`)** — スワームイベントから供給:

- 成功した `ping` → RTT観測(8サンプルの中央値ウィンドウ)。
- 失敗した `ping` → 信頼性ペナルティ。
- `identify` 受信 → アドレス(≤ 4/ピア)+ 発表されたプロトコル(kad/reqres/identify/relayビットマスク)。
- 接続確立 → 接続状態、方向(着信/発信)、経過時間、検出されたサーキットリレー。
- 接続閉鎖 → 非要請の場合のみペナルティ + チャーンカウンタ(自発的な追い出しはカウントされない)。
- 発信ダイヤル失敗 → 信頼性ペナルティ。

ビューは有界です(500ピア。それを超えると、接続中のピアのみ記憶)。

**kbucketsからの `kad_required`** — 各評価の前に、リレーはKademliaルーティングテーブルを走査します: ピアは、バケット内で `Connected`、または飽和バケット内で `Disconnected` であれば保護されます(k = 20)。保護されたピアが追い出しに選ばれることは決してありません。

**最適化ループ** — プロファイルの評価間隔で(`default` では60秒。上表参照):

1. `kad_required` を再計算(kbuckets)。
2. エンジン評価(ヒステリシス: 120秒の猶予期間、最低スコア改善 +15%、追い出しクールダウン600秒)。
3. 決定を適用: `AddCandidate` → 候補にダイヤル。`Replace` → 追い出されるピアを閉じる(一度だけ — 段階的、ループなし)、続いて置換にダイヤル。
4. **メトリクス**スナップショットがログ記録されます(INFOレベル): 既知/接続/着信/発信/サーキット/保護のカウント、平均/中央値RTT、異なるIPバケット、最大集中度、チャーン(置換/切断)。

近傍スコアは、決して信頼尺度ではありません(`scone-topology` のドキュメント参照): そこからコンセンサスの決定は派生しません。

### 到達可能性(AutoNAT)とトランスポートリレー(circuit-relay v2)

すべてのリレーがAutoNAT(クライアント + サーバー)を実行します: ノードはピアに自分への逆ダイヤルを試みてもらい、判定(`reachable`: yes = 外から到達可能、no = NATの背後、unknown = 未テスト)は5分ごとに更新され、`NodeStatus` に公開されます(フィールド `nat_reachable`、ワイヤー可変長整数 0/1/2)— CLIが表示します(`scone status`)。プライベートアドレスもカウントされます(`only_global_ips = false`): LANメッシュはLAN到達可能性をテストします。

到達可能なリレーは、circuit-relay v2 **hop**(NAT化ピアのためにトラフィックをリレーできる)と **DCUtR**(ホールパンチング: サーキット経由で接続された2ノードが直接接続を試み、リレーを解放)を有効にします。NAT化と判定されたリレーは、最初に知られている到達可能なピアに予約を自動要求し、それを通じてリッスンします(アドレス `/p2p/<relay>/p2p-circuit`)。

生成ルール: リレーされたNAT化ノードはブロック生成者になるべきではありません(レイテンシ/可用性の劣化)— PoSルール(シート = 生存ドメイン)はまだそれを禁じていません。公開testnet前に運用要件として文書化される予定です。

## トランスポートとlibp2pプロトコル

| プロトコル | 名前 | 役割 |
|---|---|---|
| QUIC (quic-v1) | — | メイントランスポート(udp) |
| identify | `/scone/id/1` | ピアアドレス発見(Kademliaテーブルを充填) |
| ping | libp2p標準 | 生存確認 |
| Kademlia | `/scone/kad/1` | DHT: キー = `DomainId`(32 B)、値 = `SignedDnsRecord` の正準エンコーディング |
| request-response | `/scone/reqres/1` | scone-protocolアプリケーションメッセージ |

### アプリケーションフレーム(`/scone/reqres/1`)

各フレームは `len: u32 BE || bytes` で、バイトは `scone-protocol` の `Message` の正準エンコーディングです(既存のコーデック — 第2のフォーマットはなし)。長さは、いかなる読み取りやアロケーションの**前に** `limits::MAX_MESSAGE_LEN` で有界です: より大きなフレームを通知するピアは、メモリコストなしに拒否されます。

交換されるメッセージ(リクエスト/レスポンス往復ごとに1つ):

| メッセージ | 方向 | 応答 |
|---|---|---|
| `Hello { version }` | ハンドシェイク | `Hello`(ローカルバージョン) |
| `Ping` / `Pong` | 生存確認 | `Pong` |
| `GetBlock { hash }` | クエリ | `Block`(RAMウィンドウ)または未知なら `Pong` |
| `GetBlocks { start, max }` | 同期 | **1つの** `Block`(高さ `start`)、なければ `Pong` |
| `Block (b)` | ブロックリレー | `Pong`(ack) |
| `Transaction (tx)` | txリレー | `Pong`(ack) |
| `GetRecord { domain_id }` | ローカルDHT | `Record`(ローカルキャッシュ)または `Pong` |
| `Record (r)` | 公開 | 受理されれば `Pong`、さもなければエラー |

ルール: **リクエストごとにちょうど1つの応答**。確認メッセージは `Pong` です。同期応答は一度に1ブロックを運びます — 追いかけは要求者側で反復します(「同期」参照)。これは意図的に有界です: ピアごとのフライト中ブロックは決して1つを超えません。

## リレー動作

### 起動

0. ネットワーク解決(`--network testnet|mainnet|devnet`、プロジェクトが開発中の間はtestnetがデフォルト — M8b)。データディレクトリはネットワークごとです(`~/.scone/<network>/`): 別のネットワークのチェーンを保持する非空のストアは拒否されます(`Corrupted`)。
1. redbストアをオープン/作成(`<data-dir>/chain.redb`)。
2. `load_chain(store, network)`: O(1)チップ + ページ化されたドメイン状態(RAMウィンドウ: ジェネシス + チップ)。
3. スワーム構築(セッションごとのephemeralなEd25519識別情報 — ネットワーク識別情報の永続化は未決定)。
4. ブートリストの構成([ピア発見](#ピア発見)参照): 明示的な `--bootstrap`(`/p2p/<peer-id>` で終わらなければならない)。そうでなければネットワークの組み込みシード、そうでなければ学習済みディレクトリ(`peers.json`)。
5. 各multiaddrにダイヤルし、Kademliaテーブルに登録(死んだダイヤルは無視 — 他がネットワークを構成します)。

### トランザクションの受信(P2PまたはCLI `submit tx` による提出)

1. 厳格な有界デコード(`MAX_MESSAGE_LEN`)。
2. 完全な暗号学的検証(`validate_transaction`: 所有者/キー結合、正準ペイロードに対する `verify_strict`)。
3. **アンチリプレイを最初に**(安価、状態非依存、決定的)、次に状態事前検査(`push_block` ルールの安価な鏡像 — 高価なPoWはプッシュ時にのみ検証されます): 再生された `RenewDomain` には嘘のPoWエラーではなく `TxReplay` が応答されます(そのプルーフは、最初の適用が超えた失効に紐付いています)。ネットワーク分離はステップ2で強制されます: 署名ペイロードはリレーのネットワークを注入し、他所で署名されたtxは `InvalidSignature` で失敗します(ワイヤー2026-09: `network` フィールドはもうなし)。`RegisterDomain` → 登録名のTLD(そうでなければ `UnknownTld`、D1/M7c)**かつオープン**(そうでなければ `TldClosed`、M8b)、次に名前が空き。`UpdateDomain` → 既知のドメイン、厳密なシーケンス `current + 1`、所有者。`RegisterTld` → 空きTLD。`TransferTld`/`RevokeTld`/`SetTldOpen` → 既知のTLD + 所有者署名者。`AssignDomain` → TLD所有者 + 空き名。`RenewTld` → 既知のプライベートTLD + 所有者、固定期間を再導出、猶予ウィンドウ事前検査。`RenewDomain` → 既知のドメイン + 所有者、次の高さでの更新PoW事前検査(現在の失効に紐付け)。
4. メモリプール挿入(固定容量、TxIdで重複排除 — プール内のtxの再提出はno-op)。
5. ピアへのブロードキャスト(送信者を除く)。

### ブロックの受信(P2P、同期、または自己生成)

1. `push_block`: 既存の完全検証(親 = 正準チップ、高さ、再計算されたMerkleルート、各トランザクションを検証してから適用 — ブロックごとにアトミック)。
2. 受理されれば: リレーは適用結果を読み(`push_block_with_gc` → ハッシュ + 決定論的GCによって除去されたドメイン)、`store_block_with_removals` で永続化します(アトミックなデルタ追加: ブロック + チップ + 変更された状態 — アップサートと除去の両方: 失効ドメイン、取り消されたTLD)。したがって、再起動が失効した登録を蘇らせることは決してありません。
3. メモリプール調整(含まれたtxは除去)。
4. ピアへのブロードキャスト(送信者を除く)。

無効なブロックは型付きエラーで拒否され、リレーは動作を続けます。ネットワークデータでパニックなし。重い適用(Argon2 + redbコミット)は `tokio::task::block_in_place` を通ります(同期ドライバー、p2p受付、生成): ランタイムは準備済みタスクを他のワーカーに移します — ブロックの適用が、DNSサービスやメトリクスを止めることはもうありません。

### DNSスナップショット(公開)

適用されたすべてのブロックの後(p2p受付 — リオルグを含む —、同期ドライバー、生成)そして起動時(再起動が、次のブロックまで生存名をNXDOMAINにしてはならない)、リレーは**権威DNSスナップショット**をDNSサーフェスが読む `watch` フィード上に公開します(完全仕様は `dns.md`):

- チップ延長(圧倒的なパス): **O(タッチ分)のデルタ**、コピーオンライト — ブロックのトランザクションがタッチされたドメインを名指しし、加えて決定論的GCの除去(`AppliedBlock.gc_removed_domains`、プッシュ時に評価)。
- リオルグ / 起動: **フル再構築**(1回のページ化 `ChainState::dns_facts` 走査)。

### ブロック生成(M5: 署名済み生成者)

`produce_interval` 秒ごと(デフォルト2秒): メモリプールが空でなければ、`BlockBuilder` がブロックを組み立てます(`timestamp = now`)。**認可された生成者**が署名し(`/docs/en/technical/blockchain.md` 参照の `header.consensus` 内ペイロード `SCONE-BLOCK-V1`)、受け取ったブロックとまったく同じ受理パスを通ります。txのないリレーは何も生成しません。

生成キー: リレーがアンカーキーで武装されているときはアンカーキー(`--anchor-key`)— アンカーは認可された生成者です — そうでなければ、生成者プールが空の間有効な固定devnetキー(オープンブートストラップ: ジェネシス)。生存するドメイン/TLDがひとたび存在すれば、devnetキーが署名したブロックは拒否されます(`InvalidProducer`)。

### アンカーループ(ファイナリティ、M5)

`--anchor-key` で(`.sconekey` キーファイル、パスフレーズは `--anchor-passphrase-env` 経由、デフォルト `SCONE_ANCHOR_PASS` — `scone-network` ライブラリのデフォルト `SCONE_ANCHOR_PASSPHRASE` には、バイナリ経由では決して到達しません。clapが常に上書きします):

- **提案者ローテーション**: 各エポックにつき、指定された委員会メンバーが**ちょうど1人**自己提案します
  (`committee[epoch % size]`、決定論的)。他のメンバーは自分のチップを決して署名しません —
  チェックポイントの内容は提案者のチップに依存し、ゲートされていない自己提案は、エポックを
  互いに競合する内容に分裂させ、各ガードがそれを正当にequivocationとして拒否しました(testnet、2026-10-21)。
- 各受理/生成ブロックの後、このノードが現在の委員会のメンバー**であり**、委員会が `MIN_FINALITY_COMMITTEE_SIZE` に達していれば、リレーは自分の `SignerGuard`(データディレクトリ内の `signer-state.dat` — 1キー = 1票/エポック、クラッシュ安全なフェイルクローズド)を通じて提案されたチェックポイントに署名し、ゴシップします。
- **受領時投票**: 内容が**私たちの**チェーン上で有効であり(参照されたブロックが同じハッシュで存在する)、委員会の署名を運ぶ提案は、私たちの署名も得ます — それがなければ、すべてのアンカーは自分のチップの内容だけに署名し、定足数は決して集まりませんでした。
- 受信した署名(`Message::Checkpoint`)は、`CheckpointData` ハッシュごとに蓄積されます(256エントリに有界)。同じデータに対して `quorum_for(委員会サイズ)` のメンバー署名が集まれば → `accept_checkpoint` + ファイナライズされたチェックポイントのブロードキャスト。
- **ファイナリティ永続化**: チェックポイントウィンドウと凍結ベースは、ファイナライズのたびにストアに書き込まれ、起動時に再読み込みされます。チェックポイントはゴシップ状態です(ブロックによって運ばれることは決してありません)— この永続化なしの再起動は空のベースから始まり、ピアから提供されたウィンドウを再びチェーンすることは決してできませんでした(ネットワークのウィンドウより下のエポックアンカーは失われています)。
- **ファイナリティ キャッチアップ**: ベースが空のリレー(新規ノードの起動)は、ベースが復元されるまで、毎ティック peer に `GetCheckpoints` を要求します。
- アンカーキーのないリレーは**受動的**のままです: 蓄積し、ファイナライズし、リレーしますが、決して署名しません。
- `GetCheckpoints` → `checkpoint_window()` ウィンドウが提供されます。
- ループのエラーはすべてログに記録されて飲み込まれます — 決して致命的ではありません。
- 状態(`GetStatus`)は `checkpoints` と `finalized_epoch` を公開します。

### フォークパーキングとリオルグ(sticky-minority修正)

同じ高さのシスターブロック、または親が現在のチップでないブロック(`ParentNotTip`/`UnknownParent`)は、決して失われません: それは**パーキング**されます(有界マップ、`PARKED_CAP = 2048`ブロック、各約KiB — 有界な最悪ケースRAM。満杯のパーキングは新規を黙って無視)。すでに `accept_block` に提供されたハッシュは重複除去されます(`seen_blocks`): 再ゴシップされた負けブロックは再評価されません。

すべての `Produce` ティックで、`reconsider_parked`:

1. 親がチップになったパーキング済みブロックの線形ドレイン。
2. パーキング済みブランチが現在のチップより強ければ(フォーク選択ルールは変更なし: ファイナリティ下限、最長チェーン、最小ハッシュ)、採用 — 変位された接尾辞自体がパーキングされます(フォークの**両側**が評価可能なまま)。

これは、等しい高さのフォークの敗者側を採用してしまったノードの固着を解きます: 一度提供されてから拒否された勝者ブロックが、永遠に忘れられるのではなく、周期的に再評価されます(chaos-30の発見: 15 ms違いの2つのシスターブロック)。

**リオルグの永続化**: 別のブランチの採用は、単調追加を通っては行けません(`NonMonotonicHeight`)。リレーは `rewind_to_height(フォーク点)` を適用し、次に `persist_state_snapshot`(チェーンのエクスポートからのライブ状態の書き換え)、次に勝った接尾辞の単調な再追加を行います(`storage.md` 参照)。永続化の失敗はログに記録され、劣化します(起動時にリプレイで治癒)— 決して致命的ではありません。

## 同期(追いかけ)

各ピアへの接続時に、リレーは `GetBlocks(start = ローカル高さ + 1)` を要求します。期待される高さをちょうど完了する各受信 `Block` が、次の高さの要求をトリガーします。枯渇するまで(`Pong` = もうなし)、または `MAX_SYNC_ROUNDS`(10,000)バッチまで — 嘘つきピアに対しても絶対的な境界です。受信ブロックは完全検証を通ります: 悪意あるピアが、無効なものを受理させることはできません。

### レコードDHT

- **put**(API/webフロー — CLIの16進チャネルは削除済み): 厳格な有界デコード(`MAX_DHT_CACHE_ENTRY`)、オンチェーン状態に対する検証(所有者、シーケンス、再計算された `record_hash` — 「レコード検証」参照)、次に `kad.put_record`(ローカル保存 + 最寄りピアへのレプリケーション、レプリケーション係数の `Quorum::Majority`)と永続redbキャッシュ。
- **get**(`domain resolve`): `kad.get_record`。待機者は自分の `QueryId` と要求された `DomainId` に紐付きます — 応答は一致するクエリのみを解決し、レコードキーは要求されたドメインと厳密に等しくなければなりません(オンチェーン検証の**前に**検査)。結果は、返される前にチェーンに対して検証されます。同時リクエストの上限: `MAX_DHT_WAITERS`(256)。超過すれば即時エラー。検証に成功した解決は、永続キャッシュに書き込まれます(M5): ローカル読み取り(`domain_info`)は、新しいDHTクエリなしでそれを提供します。
- Kademliaはサーバーモードで動作し、レコードTTLはありません(チェーンが時間ではなく有効性を統治します)。

### レコード検証(「チェーンが権威」ルール)

解決または受信された `SignedDnsRecord` が受理/返されるのは、次の場合のみです:

1. ドメインがオンチェーンで登録済みであること。
2. `record.owner` = 現在のオンチェーン所有者。
3. `record.sequence` = 現在のオンチェーンシーケンス。
4. `record_hash(record.record)`(再計算されたBLAKE3)= オンチェーンでコミットされた `record_hash`。

さもなければ: P2P側では黙った拒否、CLI側では明示的エラー。レコードのEd25519署名は、リレーによって受信時に**検証されます**(`ResolveLocal`: 所有者の公開鍵はオンチェーンで既知であり、署名が解かなければならない — 别のキーが署名したレコードは拒否、`record rejected`)。将来の検証者のためにも転送されます — オンチェーンコミットメントが主要な完全性管理のままです。

## Prometheusメトリクス

専用タスクによって提供されるHTTPエンドポイント(Prometheusテキスト形式0.0.4、手書きのエクスポジション — 新しい依存関係なし):

- デフォルトで `127.0.0.1:9101` で**公開**(CLIがそのデフォルトを適用。ライブラリのデフォルトは `None` — 1プロセスで複数のリレーが衝突することは決してない)。
- `--metrics-addr ADDR:PORT`：指定アドレスにエンドポイントをバインド（例：`127.0.0.1:9101`、公開は `0.0.0.0:9101`）。
- `--metrics-token TOKEN`: **プライベート**モードに切り替え — `Authorization: Bearer ***` 必須(定数時間比較、さもなければ `401`。トークンはログにもエクスポジションにも決して現れません)。
- バインド失敗は**エンドポイントを無効化**します(致命的でない警告)— ノードでは決してなく。

実効バインドはstdoutで通知されます(`p2p:` と同様の機械可読契約): リッスン開始と同時に `metrics: <addr>` 行。

**19カウンタ**(`counter`、単調 — ホットパス: 共有 `Arc<Metrics>` 上の `fetch_add(Relaxed)`、決してコンセンサス入力ではない):

| グループ | 系列 |
|---|---|
| チェーン | `scone_blocks_applied_total`、`scone_blocks_rejected_total`、`scone_checkpoints_signed_total`、`scone_checkpoint_sigs_received_total` |
| メモリプール | `scone_txs_admitted_total`、`scone_txs_rejected_total`、`scone_txs_evicted_grace_total`、`scone_txs_included_total` |
| DNS | `scone_dns_queries_total`、`scone_dns_cache_hits_total`、`scone_dns_cache_misses_total`、`scone_dns_local_chain_total`、`scone_dns_dht_answers_total`、`scone_dns_rejected_unverified_total` |
| DHT | `scone_dht_pulls_issued_total`、`scone_dht_negative_hits_total`、`scone_dht_negative_expired_total`、`scone_dht_negative_capped_total`、`scone_dht_cache_evictions_total` |
| ピアネットワーク | `scone_peer_connections_established_total`、`scone_peer_connections_closed_total`、`scone_peer_connections_incoming_total`、`scone_peer_dial_failures_total`、`scone_peer_blocks_received_total`、`scone_peer_transactions_received_total` |

**6ゲージ**(`scone_height`、`scone_mempool`、`scone_peers`、`scone_domains`、`scone_finalized_epoch`、`scone_addr_book_entries`)— 最初の5つは `RelayCommand` チャネル経由で15秒ごとに更新(CLI `status` と同じパス — 単一の信頼できる情報源)。`scone_addr_book_entries` はレジストリが直接維持します(アドレス帳挿入のたびに更新)。

E2Eテスト: `metrics_endpoint_public_and_private`(200 + エクスポジション、トークンなし/誤りで401、Bearerで200)。

## CLI制御(RPC→p2p統合)

JSON TCP RPCサーバーは**削除されました**: CLI(とレジストラのaxumブリッジ)は、p2pプロトコルを**直接**話します — 同じワイヤー(`/scone/reqres/1`)、リレー側の同じハンドラー、並行チャネルゼロ。各コマンドは、ノードのmultiaddr(`--peer`)へのQUIC接続を開き、1つの `Message` を交換して閉じます。

リレーが提供する制御メッセージ:

- `GetStatus` → `NodeStatus { network, nat_reachable, tip, height, domain_count, mempool, peers, finalized_epoch, checkpoints, fresh_operators }`(`scone status` が表示 — `fresh_operators` = 新鮮なハートビートを持つオペレーター、RFC proof-of-service)。
- `LookupName { name }` → `NameState`(オンチェーン状態、`lookup` と同様)。
- `GetOwnedNames { owner }` → `OwnedNames`(ある `OwnerId` の生存ドメイン + TLD)。
- `Transaction (tx)` → 自由なゴシップ付きの提出(リレーがピアに再ブロードキャスト)。
- `GetRecord { domain_id }` / `Record (r)` — DHT解決と公開(CLI側の `scone domain resolve` と同様)。
- アプリケーション拒否 → `Message::Error(msg)`(EOFではなく、読み取り可能なメッセージとしてクライアントに到達)。

クライアントは最小限のlibp2pスワーム(request-response + identify)を立ち上げ、30秒の呼び出しタイムアウト(往復20秒)。デフォルトの `--peer`:
`/ip4/127.0.0.1/udp/4001/quic-v1/p2p/12D3KooW…`(ローカル開発ノード。実際のピアIDを含む完全なmultiaddrは、リレーが `p2p:` 行でstdoutに通知します)。

内部 `RelayCommand` チャネルは、組み込みDNSサーフェスのために生き残ります(`ResolveLocal`、プロセス内の1往復、DHT待機者は決してなし — `dns.md` 参照)。

## CLI

```
scone relay [--network testnet|mainnet|devnet] [--data-dir DIR] [--listen MULTIADDR]
            [--bootstrap MULTIADDR]... [--upnp]
            [--anchor-key PATH] [--no-anchor-auto] [--anchor-passphrase-env VAR] [--genesis FILE]
            [--topology-profile tiny|small|default|high|very-high|devtest]
            [--dns ADDR] [--dns-upstream ADDR]...
            [--metrics-addr ADDR:PORT] [--metrics-token TOKEN]
scone status [--peer MULTIADDR]
scone domains [--identity ID | --pubkey HEX] [--peer MULTIADDR]
scone domain register NAME [--identity ID] [--dir DIR] [--passphrase-env VAR] [--peer MULTIADDR]
scone domain set NAME TYPE VALUE [TTL] [--identity ID] [--peer MULTIADDR]   # express single record
scone domain update NAME --file FILE [--identity ID] [--peer MULTIADDR]
scone domain get NAME [--peer MULTIADDR]
scone domain renew NAME [--identity ID] [--peer MULTIADDR]
scone domain resolve NAME [--peer MULTIADDR]

# --peer defaults to the local relay (SCONE_PEER env overrides);
# --identity defaults to the single keystore identity when there
# is exactly one. Removed (stubs suggest replacements): show,
# submit, lookup, owned.
```

すべての制御コマンドは、リレーのmultiaddrに直接QUICで話します(`--peer`、デフォルトはローカルノード — RPC→p2p統合、JSON TCPチャネルはもうなし)。リレー不在 → 明確なメッセージ「p2p node unreachable — is a relay running?」、非ゼロ終了。`--rpc` フラグは保持されますが、**廃止され無視されます**(RPCサーバーは削除済み。スクリプト互換性のみ)。

### 環境設定(`.env`)

起動時、`scone` は(dotenvy経由、clapパースの前に)カレントディレクトリから `.env` ファイルをロードします: ファイル欠落 = 黙ったno-op、決してエラーではありません。ロードされた値は、変数を宣言するオプション(`cli.rs` の `env = "…"`)の**環境ベースライン**になります。**明示的なCLIフラグが常に `.env` に勝ち**、`.env` が内蔵デフォルトに勝ちます:

```text
明示的なCLIフラグ  >  .env 変数  >  内蔵デフォルト
```

リポジトリルートの `.env.example` ファイルが、すべての変数を文書化します(`.env` にコピーして調整。`.env` は**gitignore済み** — 本物の `.env` を決してコミットしないこと)。変数:

| 変数 | 対応フラグ | 意味 |
|---|---|---|
| `SCONE_NETWORK` | `--network` | `testnet`(デフォルト)、`mainnet`、`devnet` |
| `SCONE_DATA_DIR` | `--data-dir` | データディレクトリ(デフォルト `~/.scone/<network>/`) |
| `SCONE_LISTEN` | `--listen` | P2Pリッスンmultiaddr(デフォルト: 全インターフェース、ephemeralポート) |
| `SCONE_BOOTSTRAP` | `--bootstrap` | ブートストラップシード、**反復可能**(値ごとに1行 — dotenvyが蓄積)。組み込みシードを置き換えます |
| `SCONE_GENESIS` | `--genesis` | ジェネシスファイル(創設者ノード) |
| `SCONE_MDNS` | `--mdns` | (廃止 — mDNS発見は削除されました。受け付けられるが、無視されます) |
| `SCONE_UPNP` | `--upnp` | UPnPポートマッピング(`true` = 有効) |
| `SCONE_DNS_ADDR` | `--dns` | UDP DNSサーバーのバインドアドレス |
| `SCONE_DNS_UPSTREAM` | `--dns-upstream` | 再帰フォールバック、**反復可能** |
| `SCONE_DNS_CACHE_LIMIT` | `--dns-cache-limit` | DNSキャッシュエントリ上限 |
| `SCONE_DNS_CACHE_MAX_MEM` | `--dns-cache-max-mem` | DNSキャッシュメモリ上限(例: `64MB`) |
| `SCONE_DNS_MAX_INFLIGHT` | `--dns-max-inflight` | 並行DNSクエリ上限 |
| `SCONE_METRICS_ADDR` | `--metrics-addr` | metrics endpoint address (`ADDR:PORT`)|
| `SCONE_METRICS_TOKEN` | `--metrics-token` | メトリクスエンドポイントのベアラートークン |
| `SCONE_TOPOLOGY_PROFILE` | `--topology-profile` | トポロジープロファイル(`tiny`/`small`/`default`/`high`/`very-high`/`devtest`。デフォルト `default`) |
| `SCONE_ANCHOR_KEY` | `--anchor-key` | アンカーキーファイル(チェックポイント + 生成) |

`SCONE_ANCHOR_PASS` はオプションの変数ではありません: それは**アンカーパスフレーズを保持する変数**のデフォルト名です(`--anchor-passphrase-env`、CLIレベルでデフォルト `SCONE_ANCHOR_PASS`)— リレーは、このフラグが名指しする変数からパスフレーズを読み取ります。

**アンカー自動(デフォルトON)**: `--anchor-key` がなければ、アンカーキーファイルがデータディレクトリ内に自動生成/永続化されます(`anchor.sconekey`、パスフレーズは `SCONE_ANCHOR_PASS` が定義されていればそれ、さもなければランダム生成されて `anchor.pass` に書き込まれる)— リレーは、自分のキーが生存ドメイン/TLDを所有するとすぐに生成者になります(PoS適格は決して無料ではありません)。`--no-anchor-auto` はリレーを厳密に受動的に保ちます。`--anchor-passphrase-env` のデフォルト環境変数は `SCONE_ANCHOR_PASS` です。

`--network`: デフォルトはビルドに従います(mainnet有効バイナリではmainnet、それ以外ではtestnet)。`devnet` はリセット可能なローカルネットワークです(独自の `scone-devnet` ジェネシス、ほぼゼロの0..=2ビットPoW — `~/.scone/devnet/` を消去して再起動すればチェーンはリセット)。ジェネシス、PoW難易度、デフォルトのデータディレクトリ(`~/.scone/<network>/`)を設定します。別のネットワークのtxは型付きに拒否されます(`WrongNetwork`)。

オフラインの16進サーフェス(`tx` と `submit` サブコマンド)はなくなりました(remove-tx-surface-v2)。CLIはユーザーパス(`domain register`、`domain renew`、`domains`)と、第一級コマンドとしてのM8bファミリーをカバーします: `tld open`、`tld transfer`、`tld revoke`、`domain assign`、`domain transfer`。`register` クレームは、ネットワーク難易度でPoWをマイニングします(devnetではミリ秒)。

**geoやヘルスのオプションは存在しません**: geoゾーンは**ビルドで強制**されます(組み込みDB-IP Liteテーブル — 完全な意味論は[`dns.md`](dns.md#geodns-geographic-routing)に文書化)。地理的ルーティングはオプションなしで**常に有効**であり、リゾルバーはヘルスチェックを一切行いません — 署名済みレコードセットをそのまま提供します(エンドポイントの可用性はエンドポイントオペレーターの関心事です)。すべての `--geo-*`/`--health-*` フラグはclapによって硬性拒否されます。

### `domain register|update`(M5)— ワンコマンドでの署名済みトランザクション

これらのコマンドは、構築 → 署名(keystore)→ 提出 → (確認)を単一の呼び出しに連鎖させます。

- **register**: 署名済み `RegisterDomain` を構築し(タイムスタンプ = now、devnetでは空のプルーフ)、提出し、オンチェーン確認を待ちます(シーケンス ≥ 0、30秒タイムアウト)。すでに登録された名前は、クライアント側とリレー側の両方で拒否されます。
- **update**: レコードファイルが唯一の信頼できる情報源です。シーケンス(オンチェーン + 1)と `record_hash`(ファイルのレコードの正準BLAKE3)は**導出**され、入力されません: 署名者と検証者が分岐できません。UpdateDomain確認の後、`SignedDnsRecord`(ファイルのレコード + 所有者 + 正準エンコーディングに対するEd25519署名)が、`put_record` 経由でDHTに公開されます(チェーンに対して再検証します)。未登録ドメイン、非所有者の識別情報、ファイルエラーは、チェーンに触れることなくクリーンに失敗します。

#### レコードファイル

テキスト、1行に1レコード、`#` = コメント、空行は無視。ソート済み正準エンコーディングにより、ファイルの順序はコミットされるハッシュに影響しません。ファイルは完全なセットです(更新は置換であって、マージではありません)。型:

```text
A 192.0.2.1
AAAA 2001:db8::1
CNAME www.example.uip
NS ns1.example.uip
MX 10 mail.example.uip
TXT free text (spaces normalized)
```

型付きエラー: 未知の型、無効なIP/名前(行番号付き)、空セット、重複、> `MAX_RECORDS_PER_SET`。

## `--genesis`(創設者)

`scone relay --genesis genesis.bin` — **創設者ノード専用**(キックスターターキャンペーン。ファイルは `scone-genesis` バイナリが生成)。規範ルール:

1. ファイルは、1 MiBを超えれば**読み取り前に**拒否され、次に厳格にデコードされます(`decode_complete`: 後続バイトなし)。ブロック0のハッシュは**再計算**され、ファイルからは読まれません。
2. 意味的検証: 高さ0、`consensus` が要求されたネットワークのネットワークIDで開始(`--network`)、トランザクションなし、正準アロケーションマニフェスト。違反はすべて、起動時の型付きエラーです(決してパニックではなく、デーモンは起動しません)。
3. **無垢の**データディレクトリ: チェーンはアロケーション済みジェネシスの上に生まれます — アロケーションは、いかなるディスク書き込みの**前に**RAMで適用され、次に永続化されます(シードされた状態 + メタキーの下のジェネシスブロック)。したがって、最初に生成されるブロックは、アロケーション済みジェネシスを延長することしかできません。
4. **すでに生産的な**データディレクトリ: ストアの誕生ジェネシスは、ブロック1の `prev_hash` です(自己認証)。等しいファイルハッシュ = 黙ったOK(このオプションは**最初の起動以降は冗長**)。異なれば = 型付き拒否 — ノードは `--genesis` 経由ではなく、同期を通じてネットワークに参加します。
5. オプション**なし**の再起動: 創設者生まれのデータディレクトリは、メタキー経由でアロケーション済みジェネシスを復元します(毎回の起動時にRAMで再バインド)。

定数ジェネシハッシュのピン留め(ネットワーク分岐防止)は、別の作業項目です。

### `scone-genesis` バイナリマニフェスト(キックスターター)

`scone-genesis` バイナリ(クレート `scone-genesis`)は `genesis-manifest.json` マニフェスト(`ALLOC-V1` 形式)を準備し、次に `genesis.bin` を生成します。規範ルール:

- `scone-genesis new --network <network>` が空のマニフェストを作成します。`add`/`remove`/`list` がアロケーションを管理し、`build` が `genesis.bin`(+ ハッシュ)を生成し、`verify` がそれを再検証します。
- `add domain <name>` は、ドメインの**TLDがすでにマニフェスト内でアロケーション済み**であることを要求します(そうでなければ即時拒否 — チェーンはジェネシス適用でそれを拒否していたでしょう)。その `valid_until` は**自動**です: `born_at + 1年`(マニフェストのdefault_validity)。手入力は決してありません。
- `add tld <name>`: **デフォルトでオープン**(誰のものでもない、セルフサービス、1年の下限、ワイヤー上でowner = ゼロ — 種別 `0x03`)。`--private` = プライベート(所有者、ジェネシス期間 `born_at + TLD_TERM_SECS` — 3年、標準ライフサイクル: `RenewTld`、60日猶予)。
- 名前による正準ソート(等しい内容 → 同じブロック → 同じハッシュ、挿入順序に関係なく)。重複は拒否。

## リミットとメモリ保証

| リソース | 境界 |
|---|---|
| P2Pフレーム | パース前の `MAX_MESSAGE_LEN`(1 MiB) |
| メモリプール | 4096 tx(超過は型付き拒否)。ドメインごとの保留tx `MAX_PENDING_PER_DOMAIN = 4` |
| メモリプール猶予 | 非決定的な理由でフラッシュされたtxの再挿入 `MAX_GRACE_ATTEMPTS = 10` 回(40秒/サイクルで約7分)、その後追い出し |
| フォークパーキング | `PARKED_CAP = 2048` 競合ブロック |
| 蓄積チェックポイント | `MAX_PENDING_CHECKPOINTS = 256`(追い出し: 最小データハッシュ) |
| 同時DHT待機者 | `MAX_DHT_WAITERS`(256)。DHTルックアップは30秒に有界(`DHT_LOOKUP_TIMEOUT`) |
| 同期応答ごとのブロック | 往復ごとに1。ピアごと ≤ 10,000バッチ(`MAX_SYNC_ROUNDS`)。リクエストのプロトコル境界 `MAX_BLOCKS_PER_REQUEST = 128` |
| DHTレコード | `MAX_DHT_CACHE_ENTRY`(64 KiB) |
| 永続DHTキャッシュ(ディスク) | 二重LRU境界: `DHT_CACHE_MAX_ENTRIES`(10万エントリ)かつ `DHT_CACHE_MAX_BYTES`(2 GiB)— 最初に達した方がキー順で追い出し。GC追い出しはブロックと同じトランザクション内(チェーンGCに殺されたドメインが、孤児エントリを後に残すことは決してない) |
| DHTネガティブキャッシュ | `DHT_NEGATIVE_TTL_SECS`(30秒)、FIFO上限 `DHT_NEGATIVE_CAP`(4,096) |
| RAM内チェーン | ジェネシス + チップウィンドウ(履歴はストアから提供) |

アンチループルール(H2): トランザクションは、メモリプールに**実際に挿入**された場合にのみ再ブロードキャストされます(初見)。重複(すでにプール内またはチェーン内)は決して再リレーされません。すでに正準なブロックは、`push_block` の前に検出され無視されます(再保存なし、再ブロードキャストなし)。

未決定事項: リレーの永続的なネットワーク識別情報、ブロードキャストゴシップ(現在: 既知の各ピアへのユニキャストrequest-response)。初稿以降に解決済み: 本物のフォーク選択(パーキング + reconsider_parked、上記参照)、PoWコンセンサス(署名済み生成者 + アンカー委員会)、DNSリゾルバー(`--dns` サーフェス、dns.md参照)、制御チャネル(RPC→p2p統合: JSON TCPはもうなし)。

## グレースフルシャットダウン(SIGTERM / SIGINT)

`scone relay` デーモン(フォアグラウンド)は、`tokio::signal` 経由でSIGTERMとSIGINTを捕捉します: シグナル時、`Relay::run` フューチャーはキャンセルされ(バイナリ内の `tokio::select!`)、libp2p Swarmをドロップし、QUICソケットを閉じます。続いてtokioランタイムは、有界なシャットダウン(≤ 2秒)で停止し、デタッチされたDNSタスク(UDP/TCP)をドロップし、そのソケットを閉じます — 目標は、`kill` が孤児のUDPソケットを残さないことです(testnet 60で観測されたケース)。シグナル駆動のシャットダウンは `relay shutting down (signal)` をログに記録し(INFO、stderr)、終了コード **0** で終了します: 要求されたシャットダウンは失敗ではありません。ハンドラー設置に失敗した場合、シグナルのデフォルト動作が適用されます(劣化、決してパニックではない)。


## Torトランスポート(未解決問題)

QUIC/UDPは平文のSOCKSプロキシを越えられないため、リレーは今日、Tor上で自明には動かせません。ネイティブなlibp2p Torトランスポート(オニオンアドレスのリレー)が追跡されている前進 path です — 現状と `.tor` CNAMEブリッジ(ヒドゥンサービス向けの可読な名前、本日サポート済み)については、ウェブサイトの検閲抵抗ガイドを参照してください。
