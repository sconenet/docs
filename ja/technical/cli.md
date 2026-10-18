# CLI(`scone`)— ロギングと詳細度

> `scone` バイナリのロギングの規範的動作。ドキュメントとコードの不一致はバグです(信頼できる正しい情報源を修正すること)。

## 原則

- **stdoutはコマンド出力専用**です(`scone status`、`identity list`、`status` など)。スクリプトはstdoutをリダイレクトしても、その中にログが混入することは決してありません。パイプは信頼性を保ちます。認められた例外:`scone relay` は、リスナーがバインドされた時点でデータ行 `p2p: <multiaddr>`、`dns: <addr>`、`metrics: <addr>` をstdoutに出力します(`--bootstrap` / 設定で使えるデータであり、ログではありません。各行ちょうど1回、即時にフラッシュされます)。
- **すべてのログはstderrに出力されます**(コンパクトな `tracing` フォーマット)。リレーのログも含みます。リレーは移行前からstderrにログを出力していました。
- CLIのユーザー向けエラーメッセージ(`scone: <error>`)はstderrのままです。これはUXであり、ログではありません。

## レベルと `--verbose` / `-v`

| 起動方法 | 有効レベル |
|---|---|
| `scone <cmd>`(`-v` なし) | `warn` |
| `scone relay`(`-v` なし) | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …`(それ以上も) | `trace` |

`-v`/`--verbose` は**グローバルかつ反復可能**なフラグです。サブコマンドの前でも後でも受け付けられます(`scone -vv identity list` と `scone identity list -vv` は等価です)。

**`scone relay` がデフォルトでINFOのままの理由**:リレーはデーモンであり、運用可能であるためには、フラグを要求することなく、自分の活動(RPC/p2pのリッスン、受け入れて生成したブロック、接続したピア、拒否)をログに記録しなければなりません。一方、一回きりのコマンドは、ターミナルやスクリプトを汚さないように、黙して(WARNのまま)動作します。

## `RUST_LOG`(上書き)

`RUST_LOG` 環境変数が設定されている場合、`-v` から導かれるレベルを**完全に置き換えます**(`env-filter` 構文):

```bash
RUST_LOG=debug scone identity list        # -v なしで debug
RUST_LOG=error scone -vv relay            # -vv より error が優先
RUST_LOG=scone_network=trace,info scone relay   # ターゲットごと
```

## 環境設定(`.env`)

引数をパースする前に、`scone` はカレントディレクトリから `.env` ファイルをロードします(`main.rs` の `dotenvy::dotenv()`)。ファイルが存在しなくても、黙って何もせず、エラーにはなりません。ロードされた値は、環境変数を宣言するリレーオプション(`cli.rs` の `env = "…"`)のベースラインになります。

優先順位(1つのオプション、3つのソース):

```text
明示的なCLIフラグ  >  .env 変数  >  内蔵デフォルト
```

- コマンドラインで渡されたフラグが常に優先されます。
- そうでなければ変数(`.env` からロードされたものも、実際のプロセス環境に設定されたものも、同じ変数、同じ効果)。
- そうでなければclapの内蔵デフォルト。

反復可能な変数(`SCONE_BOOTSTRAP`、`SCONE_DNS_UPSTREAM`)は蓄積されます。`.env` に値ごとに1行(dotenvyが追記)。

`SCONE_*` 変数の完全な表(ネットワーク、P2Pリッスン、ブートストラップ、ジェネシス、mDNS、UPnP、DNSサーフェス、メトリクス、トポロジープロファイル、アンカー)とそれぞれの意味は[`relay.md`](relay.md#environment-configuration-env)にあります。コメント付きテンプレートはリポジトリルートの `.env.example` です(`.env` にコピーして使います。コミットは決してしないこと。`.env` はgitignoreされています)。

## レベルの意味(コード規約)

- `error!`:失敗(現在の呼び出し箇所はなし。ネットワークの失敗は `warn!` として飲み込まれ、致命的なローカルの失敗だけが `run()` からの型付きエラーとして表面化します)。
- `warn!`:拒否された敵対的または悪意のあるネットワーク入力、メモリプールから追い出された古いtx、生成中に破棄された競合tx、アウトバウンド/インバウンドの失敗、DHTキーの不一致。
- `info!`:ブロックの受理/生成、RPC/p2pのリッスン、ピア接続、リレー起動(状態チェックポイント)。
- `debug!`:内部詳細(tx受理、ピア切断、RPCラウンドトリップ、キーストア一覧、`SCONE_KAD_DEBUG` 配下のkadイベント)。
- `trace!`:ペイロード用に予約(現在の呼び出し箇所はなし)。

ライブラリクレート(`scone-network` など)は `tracing` マクロを出力しますが、**サブスクライバを初期化しません**:初期化は `scone` バイナリ内にだけ存在します(サブコマンドディスパッチの前。リレーは起動時からログを必要とします)。


## 簡単な使い方(cli-simplify-v1)

- **`--peer` は不要**:コマンドはデフォルトでローカルリレーと対話します。`SCONE_PEER` 環境変数または `--peer` で上書きできます。
- キーストアが単一の識別情報しか保持していないとき、**`--identity` は不要**です(黙って使用されます。複数ある場合は、エラーがそれらを一覧表示します)。
- 削除済み(stubが代替を示します):`show`、`submit`、`lookup`、`owned`。
- 新規:`scone domain renew <name>`、`scone domains`、`scone domain get|set`(単一レコード、ファイル不要)。

```bash
scone identity generate me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone domain renew example.scone
```

## TLDライフサイクル(cli-unified-lifecycle)

`scone tld register <tld>` は、`scone domain register` と対称な、ワンコマンドのTLD入口です。オンチェーンの状態を最初に読み、それから適切なアクションが実行されます:

- **空きTLD** → 登録され、デフォルトで**OPENとして**誕生(フラグなし:セルフサービスの名前空間、PoWは1/3)。`--private` を指定すると、フル価格のPoWで所有する名前空間を選択できます(明示的な確認を伴う経済的判断)。`--yes` は確認をスキップします(スクリプト用)。
- **あなたのもので、90日以内に失効** → 更新が提案され(現在の失効時刻に紐づくPoW — 再利用防止)、確認後に実行されます。
- **あなたのもので、失効まで余裕がある** → 何もすることはない(強制するなら `tld renew --force`)。
- **他人のもの** → 所有者を示すクリーンな型付きエラー。

```bash
scone tld register mytld            # open として誕生(デフォルト)
scone tld register mytld --private  # 所有する名前空間(確認あり)
scone tld renew mytld               # 明示的な更新
```

同様に、所有している名前に対する `scone domain register <name>` は、90日以内に失効する場合、失敗する代わりに更新を提案します。

## 高度なコマンド

上記のデフォルトは日常の使用をカバーします。スクリプトやWeb/API相互運用のために、完全なサーフェスは引き続き利用可能です:

すべての操作には正規のパスが1つだけ存在します。かつてのルートエイリアス（`show`、`lookup`、`owned`、`renew`）は削除されました（remove-cli-aliases）。実在しなかった `domain info` の記載も同時に削除されています。

オフラインの16進サーフェス(`tx build|sign|verify`、`submit tx` サブコマンド)は削除されました(remove-tx-surface-v2)。これらが担っていたM8bの操作は、今や第一級のコマンドです。`scone tld open <tld>`(一方向のセルフサービスオープン、オープン用PoWは代わりにマイニングされます)、`scone tld transfer <tld> <owner>`、`scone tld revoke <tld>`、`scone domain assign <name> <owner>`(TLD所有者、プライベートTLD)、そして `scone domain transfer <name> <owner>`。いずれも:状態読み取り、署名、送信、txid。DNSレコードの解決は `scone domain resolve <name>`(旧 `record get`)に移動しました。16進公開チャンネル(`record put`)はなくなりました。編集は `domain set` / `domain update` 経由で行います。
- `scone domain export <name> [--out FILE]` — チェーン検証済みのレコードセットを、ちょうど `update --file` のフォーマットで書き出します(往復保証:export → 編集 → update が同じセットを再送信)。`--out` なしの場合はstdout。`domain resolve` は同じ可読フォーマットを使うので、`resolve > zone.txt` はそのままインポート可能です。テキスト構文のない種類(GEOROUTE、不明)はコメント付き警告としてエクスポートされます。黙って破棄されることは決してありません。

`domain get` は `domain_id` と `valid_until` も表示します。

## ビルド:dev対mainnet有効(mainnet-compile-gate)

デフォルトのビルドには**mainnetが含まれません**。`scone relay --network mainnet`(および `SCONE_NETWORK=mainnet` のレジストラサービス)は、再ビルドのヒントとともに即座に失敗します。公式配布バイナリは次でビルドされます:

```bash
cargo build --release --features mainnet
```

`scone --version` はそのようなビルドでは `+mainnet` を表示し、リレーの起動ログは `build: mainnet-enabled` と出力します。

デフォルトの `--network` もビルドに従います。mainnet有効ビルドでは `mainnet`(公式バイナリはユーザーを本番ネットワークに導きます)、それ以外では `testnet` です。明示的な `--network` フラグや `SCONE_NETWORK` 環境変数が常に優先されます(clapの順序:フラグ > 環境変数 > デフォルト)。これは運用上のガードレール(うっかり実行、dev/CIバイナリ)であり、暗号論的な境界ではありません。敵対的なノードに対する防御は、コンセンサスルールが担い続けます。
