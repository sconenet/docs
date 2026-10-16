# Keystore — `.sconekey` キーファイルフォーマット仕様

> 規範文書。本文書と `crates/scone-keystore/src/lib.rs` の不一致はバグです(信頼できる正しい情報源を特定し、修正すること)。実装参照: クレート `scone-keystore`(v2)。

## 目的

Ed25519の秘密鍵(*シード*、32バイト)を、パスフレーズで暗号化してディスクに保存します。キーが平文でディスクに触れることは**決して**ありません。すべての秘密(パスフレーズ、導出物)は使用後にメモリ上でゼロ化されます。

## 概要

- **KDF**: Argon2id(v0x13)、デフォルトパラメータ `m_cost` = 64 MiB、`t_cost` = 3、`p_cost` = 1(`argon2` クレートのデフォルト)。
- **暗号化**: XChaCha20-Poly1305(AEAD)、KDFから導出された32バイトキー、ファイルごとの24バイトランダムnonce。
- **暗号化対象**: ちょうどEd25519シード(32バイト)。Poly1305タグ(16バイト)は暗号文の接尾辞として保存されます(AEAD `encrypt` APIの標準出力)。

## バイトの封印(`bytes` モジュール、web)

`scone_keystore::bytes::{seal_to_bytes, open_from_bytes}` は、ファイルシステムに触れずに同じフォーマットを生成・消費します。Webクライアントはブラウザ内で識別情報を生成し(`scone-crypto` + bip39)、ちょうどのキーファイルバイトを封印します(ダウンロード可能、CLIがそのまま開ける — スイートに相互運用テストあり)。ネイティブパス(`create`/`create_with_phrase`)は同じ封印処理に委譲します。フォーマットの一つのエンコーディング、三つの行き先。

## バイナリフォーマット(v2)— 52バイトヘッダー + 公開メタデータ + 暗号文

```text
Offset  Size        Field
0       9           magic: "SCONEKEY1" (ASCII — unchanged since v1)
9       1           version: u8 = 2
10      1           m_cost_mib: u8 (m_cost in MiB; m_cost_kib = value × 1024)
11      1           t_cost: u8 (Argon2id iterations)
12      16          Argon2id salt (random per file)
28      24          XChaCha20-Poly1305 nonce (random per file)
52      32          public_key: Ed25519 public key (32 bytes, IN THE CLEAR)
84      8           created_at: u64 big-endian (Unix, seconds)
92      1           label_len: u8 (0..=255)
93      label_len   label: UTF-8 (readable without the passphrase, may be empty)
—       48          ciphertext (32 bytes encrypted seed) || Poly1305 tag (16 bytes)
```

`public_key`/`created_at`/`label` メタデータは公開です。パスフレーズ**なし**で読み取り可能です(`public_metadata_from_bytes`)。公開鍵は性質上公開のものです。ラベルは単なる表示名です。隠しても何も得られず、パスフレーズなしの一覧の利点を失います。合計長はラベルに依存します。空ラベルで141バイト(KEYFILE_LEN + 32 + 8 + 1)、ラベルの長さを加算。

**v1(廃止、ちょうど100バイト)**: magic ‖ version=1 ‖ m ‖ t ‖ salt ‖ nonce ‖ 暗号文 — 公開鍵は保存されていません(ロックされたファイルは匿名でした)。v1ファイルは引き続き**読み取り可能**です(メタデータは `None`。`identity show` にはパスフレーズが必要。`scone identity restore` による再封印でv2に移行)。

### 設計上の選択

- **ファイル内のKDFパラメータ**: `m_cost_mib` と `t_cost` はヘッダーから読み取られます(読み取り時にハードコードされない)。将来のバージョンで引き上げても既存ファイルを壊しません。
- **読み取り時のセキュリティ境界**: `m_cost_mib` > 64、または `t_cost` = 0 もしくは > 10 を宣言するファイルは、いかなるアロケーションの前にも拒否されます(`Error::InvalidParams`)。悪意あるファイルが、ノードにギガバイトのアロケーションや無限ループを強制できません。書き込み時のデフォルトは `m_cost_mib` = 64、`t_cost` = 3。
- **明示的なmagic + バージョン**: キーファイルでないファイル(`Error::InvalidFile`)や、未サポートのバージョンのファイル(`Error::UnsupportedVersion`)は、パニックなしにクリーンに拒否されます。
- **誤ったパスフレーズ**: Poly1305認証の失敗は、なぜ失敗したかを示しません。誤ったパスフレーズと破損した暗号文は区別できず、同じ明確なエラー `Error::WrongPassphrase` として表面化します。

## 暗号化キーの導出

```text
key = Argon2id(passphrase, salt, m_cost = m_cost_mib × 1024 KiB,
               t_cost = t_cost, p_cost = 1, output = 32 bytes)
```

続いて:

```text
ciphertext || tag = XChaCha20-Poly1305-Seal(key, nonce, plaintext = seed)
```

v1に関連データ(AAD)はありません。ファイルごとにランダムな24バイトのnonceにより、nonce再利用のリスクは無視できます(キーファイルは一度しか書かれません)。

## API(`scone-keystore`)

| 関数 | 役割 |
|---|---|
| `create(path, passphrase) -> GeneratedKey` | キーを生成し、暗号化されたキーファイルを書き込む(親ディレクトリを作成)。既存ファイルの上書きは**拒否**(`Error::KeyfileExists`)。キーをメモリ上で返す |
| `create_overwriting(path, passphrase) -> GeneratedKey` | 同じだが、既存のキーファイルを明示的に置き換える |
| `open(path, passphrase) -> SigningKey` | 復号して署名キーを返す |
| `list(dir) -> Vec<KeyEntry>` | ディレクトリ内の `*.sconekey` ファイルを一覧(名前順)。ディレクトリなしなら空リスト |

`GeneratedKey.signing_key` は `ed25519-dalek` によってドロップ時にゼロ化されます(デフォルトの `zeroize` 機能)。導出されたAEADキー、シード、復号された平文は `zeroize::Zeroizing` に包まれます。ドロップは、`?` で伝播するエラーを含むすべての出口パスをカバーします。

## ファイル権限

Unixでは、キーファイルはプロセスのumaskにかかわらず、権限 `0600`(所有者のみ読み書き)で作成されます。`OpenOptions` + `set_permissions` 経由です。存在チェックと作成はアトミックです(`create_new`、`O_EXCL`)。同時並行の2つの生成が、黙って互いを上書きすることはできません。

## CLI

```bash
scone identity generate <n> [--dir <directory>] [--passphrase-env VAR] [--force]
scone identity list [--dir <directory>]
scone identity show <n> [--dir <directory>] [--passphrase-env VAR]
```

- デフォルトディレクトリ: `$HOME/.scone/keys`。
- 識別情報名は厳格に検証されます: 空、`.`、`..`、`/`、`\\`、NUL、先頭のドット、`.sconekey` 接尾辞は拒否されます(名前はキーディレクトリ内のファイル名になります。パス要素がkeystoreを脱出することはできません)。
- `--passphrase-env` がない場合、パスフレーズはターミナル上で**マスク入力**で要求されます(`rpassword`: エコーなし。`generate` では確認付きで2回)。
- `identity generate` は **BIP39リカバリーフレーズ(24語)** をちょうど1回だけ表示します — 新鮮な256ビットエントロピー、語だけで同じキーを再生成できます(`scone identity restore`)。どこにも保存されません。
- `identity generate` は既存キーファイルの上書きを**拒否**します。`--force` が明示的に置き換えます(新しいsalt/nonce、新しいキー、新しいフレーズ)。
- `identity list` は、v2ファイルについて、パスフレーズ**なし**で公開鍵とラベルを表示します(公開メタデータ)。
- `identity show` はv2キーファイルの公開メタデータを読み、パスフレーズ**なし**で、公開鍵、小文字16進の `OwnerId`(64文字、`DomainId` と同じ規約)、ラベル、作成日を表示します。v1ファイル(メタデータなし)は引き続きパスフレーズを要求します。

## このフォーマットが意図的にしないこと

- 1ファイルに複数キーなし(1ファイルに1シード)。
- パスフレーズ変更なし(将来来るとしても: 新しいsalt/nonceでファイルを書き直す)。


## デフォルト識別情報(cli-simplify-v1)

署名識別情報を必要とするコマンド(`domain register`、`domain set`、`domain update`、`domain renew`)は、識別情報がちょうど1つのとき、`--identity` なしでkeystoreの単一識別情報を使います。複数ある場合、コマンドは失敗し、それらを一覧表示します(`scone identity list`)。`--identity <name>` を明示的に渡してください。
