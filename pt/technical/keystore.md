# Keystore — especificação do formato de arquivo de chaves `.sconekey`

> Documento normativo. Qualquer divergência entre este documento e
> `crates/scone-keystore/src/lib.rs` é um bug (identificar a fonte
> da verdade, corrigi-la). Implementação de referência: crate
> `scone-keystore` (v2).

## Objetivo

Armazenar uma chave privada Ed25519 (a *seed*, 32 bytes) em disco,
criptografada com uma passphrase. A chave **nunca** toca o disco em
texto claro; todos os segredos (passphrase, material derivado) são zerados
em memória após o uso.

## Visão geral

- **KDF**: Argon2id (v0x13), parâmetros padrão `m_cost` = 64 MiB,
  `t_cost` = 3, `p_cost` = 1 (padrões da crate `argon2`).
- **Criptografia**: XChaCha20-Poly1305 (AEAD), chave de 32 bytes derivada
  do KDF, nonce aleatório de 24 bytes por arquivo.
- **Conteúdo criptografado**: exatamente a seed Ed25519 (32 bytes); a
  tag Poly1305 (16 bytes) é armazenada como sufixo do ciphertext
  (saída padrão da API `encrypt` do AEAD).

## Selagem de bytes (módulo `bytes`, web)

`scone_keystore::bytes::{seal_to_bytes, open_from_bytes}` produz
e consome o MESMO formato sem tocar o filesystem: o
cliente web gera a identidade no navegador (`scone-crypto` +
bip39) e sela os bytes exatos do arquivo de chaves (baixável, abrível
pela CLI tal qual — teste de interoperabilidade na suíte). O caminho
nativo (`create`/`create_with_phrase`) delega à mesma
selagem: uma única codificação do formato, três destinos.

## Formato binário (v2) — header de 52 bytes + metadados públicos + ciphertext

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

Os metadados `public_key`/`created_at`/`label` são públicos: legíveis
**sem a passphrase** (`public_metadata_from_bytes`). A
chave pública é pública por natureza; o label é apenas um nome de
exibição — escondê-lo não ganha nada e custa a listagem sem passphrase. O
comprimento total depende do label: 141 bytes com label vazio
(KEYFILE_LEN + 32 + 8 + 1), mais o comprimento do label.

**v1 (substituída, exatamente 100 bytes)**: magic ‖ version=1 ‖ m ‖
t ‖ salt ‖ nonce ‖ ciphertext — a chave pública NÃO era armazenada (um
arquivo travado era anônimo). Arquivos v1 permanecem **legíveis**
(metadados `None`, passphrase exigida para `identity show`;
re-selagem via `scone identity restore` os migra para v2).

### Escolhas de design

- **Parâmetros do KDF no arquivo**: `m_cost_mib` e `t_cost` são
  lidos do header (não fixos no código em tempo de leitura), permitindo
  elevá-los em versões futuras sem quebrar arquivos existentes.
- **Limites de segurança em tempo de leitura**: um arquivo declarando
  `m_cost_mib` > 64 ou `t_cost` = 0 ou > 10 é rejeitado
  (`Error::InvalidParams`) *antes* de qualquer alocação — um arquivo
  malicioso não pode forçar um nó a alocar gigabytes ou entrar em loop
  indefinidamente. O padrão em tempo de escrita é `m_cost_mib` = 64,
  `t_cost` = 3.
- **Magic + versão explícitos**: um arquivo que não é um keyfile
  (`Error::InvalidFile`) ou de versão não suportada
  (`Error::UnsupportedVersion`) é rejeitado de forma limpa, sem
  pânico.
- **Passphrase errada**: a falha de autenticação Poly1305 não
  indica *por que* falhou; uma passphrase errada e um ciphertext
  corrompido são portanto indistinguíveis e surgem como o mesmo erro
  distinto `Error::WrongPassphrase`.

## Derivação da chave de criptografia

```text
key = Argon2id(passphrase, salt, m_cost = m_cost_mib × 1024 KiB,
               t_cost = t_cost, p_cost = 1, output = 32 bytes)
```

Depois:

```text
ciphertext || tag = XChaCha20-Poly1305-Seal(key, nonce, plaintext = seed)
```

Sem dados associados (AAD) na v1. O nonce de 24 bytes, aleatório por
arquivo, torna o risco de reuso de nonce desprezível (arquivos de chaves
são escritos apenas uma vez).

## API (`scone-keystore`)

| Função | Papel |
|---|---|
| `create(path, passphrase) -> GeneratedKey` | Gera uma chave, escreve o keyfile criptografado (cria diretórios pais); **recusa sobrescrever** um arquivo existente (`Error::KeyfileExists`); retorna a chave em memória |
| `create_overwriting(path, passphrase) -> GeneratedKey` | O mesmo, mas substitui explicitamente um keyfile existente |
| `open(path, passphrase) -> SigningKey` | Descriptografa e retorna a chave de assinatura |
| `list(dir) -> Vec<KeyEntry>` | Lista os arquivos `*.sconekey` num diretório (ordenados por nome); diretório ausente = lista vazia |

`GeneratedKey.signing_key` é zerado no drop por `ed25519-dalek`
(feature `zeroize` padrão). A chave AEAD derivada, a seed e o
texto claro descriptografado são envoltos em `zeroize::Zeroizing`:
o Drop cobre todos os caminhos de saída, incluindo erros propagados por
`?`.

## Permissões de arquivo

No Unix, o keyfile é criado com permissões `0600`
(leitura/escrita apenas pelo owner), via `OpenOptions` +
`set_permissions` — independentemente do umask do processo. Existência e
criação são atômicas (`create_new`, `O_EXCL`): duas gerações
concorrentes não podem silenciosamente sobrescrever-se.

## CLI

```bash
scone identity generate <n> [--dir <directory>] [--passphrase-env VAR] [--force]
scone identity list [--dir <directory>]
scone identity show <n> [--dir <directory>] [--passphrase-env VAR]
```

- Diretório padrão: `$HOME/.scone/keys`.
- O nome da identidade é estritamente validado: vazio, `.`, `..`, `/`,
  `\\`, NUL, ponto inicial e o sufixo `.sconekey` são rejeitados
  (o nome torna-se um nome de arquivo dentro do diretório de chaves —
  nenhum componente de caminho pode escapar do keystore).
- Sem `--passphrase-env`, a passphrase é pedida no
  terminal **com entrada mascarada** (`rpassword`: sem eco; duas vezes com
  confirmação para `generate`).
- `identity generate` **exibe a frase de recuperação BIP39 (24
  palavras)** exatamente uma vez — entropia fresca de 256 bits, as
  palavras sozinhas regeneram a mesma chave (`scone identity restore`);
  elas nunca são armazenadas em lugar nenhum.
- `identity generate` **recusa sobrescrever** um keyfile
  existente; `--force` o substitui explicitamente (novo salt/nonce, nova
  chave, nova frase).
- `identity list` mostra, para arquivos v2, a chave pública e o
  label **sem a passphrase** (metadados públicos).
- `identity show` lê os metadados públicos do keyfile v2 e
  exibe a chave pública, o `OwnerId` em hex minúsculo (64
  caracteres, mesma convenção que `DomainId`), o label e a
  data de criação **sem a passphrase**; um arquivo v1 (sem
  metadados) ainda a pede.

## O que este formato deliberadamente não faz

- Sem múltiplas chaves por arquivo (uma seed por arquivo).
- Sem troca de passphrase (se um dia vier: reescrever o arquivo com um
  novo salt/nonce).


## Identidade padrão (cli-simplify-v1)

Comandos que precisam de uma identidade assinante (`domain register`,
`domain set`, `domain update`, `domain renew`) usam a identidade ÚNICA
do keystore sem `--identity` quando há exatamente uma.
Com várias identidades o comando falha e as lista
(`scone identity list`); passe `--identity <nome>` explicitamente.
