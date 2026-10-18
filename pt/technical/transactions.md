# Transações assinadas (formato v1)

Documento de referência **normativo** do formato das transações assinadas.
Ele especifica os campos, a ordem exata do wire, os tamanhos, o payload
assinado e as regras de validação, de modo a permitir uma implementação
independente sem ler o Rust. A implementação de referência
é [`scone-protocol`](../../../crates/scone-protocol) (codificação) e
[`scone-blockchain`](../../../crates/scone-blockchain) (validação).

`PROTOCOL_VERSION` = **1**. Existe apenas um conjunto de formatos: qualquer
byte de versão diferente de `0x01` (numa transação) ou diferente da
versão local (num bloco) é rejeitado com `UnsupportedVersion`.

## Visão geral

```text
Transaction = disc u8 || version u8 || payload
```

| Campo | Tamanho | Papel |
|---|---|---|
| `disc` | 1 B | tipo da transação (tabela abaixo) |
| `version` | 1 B | **exatamente `0x01`** (versão do formato tx) |

**As transações não carregam mais um campo `network` no wire**
(mudança de 2026-09): o `network_id` (`str` ≤ 16, ASCII `[a-z0-9-]` —
por ex. `scone-testnet`, 13 bytes) vive **apenas no payload de assinatura**
— é injetado lá como primeiro campo pelo assinante (ver [Payload
assinado](#payload-assinado)) — e permanece por rede no digest PoW.
Uma transação **assinada** para uma rede portanto nunca verifica em
outra: sua assinatura falha (`InvalidSignature`) contra o payload
recomputado pela chain, que injeta sua própria rede (ver
`/docs/en/technical/blockchain.md` — redes e gênese).

### Discriminantes (tabela normativa)

Cada discriminante é uma **constante opaca arbitrária fixa**: os
valores são deliberadamente distintos, para que o wire **nunca**
sugira uma ordem lógica entre os tipos (sem sequência
0x01/0x02/0x03…). `0x00` **nunca** é um tipo válido (byte típico de
corrupção). Tipos futuros tomarão qualquer valor livre não usado,
documentado de uma vez por todas nesta tabela.

| Discriminante | Tipo | Papel |
|---|---|---|
| `0x21` | `RegisterDomain` | reivindicação de um domínio |
| `0x52` | `UpdateDomain` | nova versão dos dados DNS de um domínio |
| `0x93` | `RegisterTld` | reivindicação de um domínio de topo (M7) |
| `0x47` | `TransferTld` | transferência de um TLD a um novo owner (M8a) |
| `0x6B` | `RevokeTld` | abandono de um TLD (M8a) |
| `0xB8` | `SetTldOpen` | abertura **irreversível** de um TLD ao auto-registro (sentido único, owner-option; `open=false` é rejeição tipada) |
| `0xD4` | `AssignDomain` | atribuição direta de um domínio pelo owner do TLD (M8a) |
| `0x3C` | `RenewDomain` | extensão do registro de um domínio (M8a; carrega o nome canônico desde a owner-option — um TLD aberto segue a expiração) |
| `0x8F` | `RenewTld` | renovação do mandato de 3 anos de um TLD PRIVADO (PoW igual à criação) |
| `0x15` | `SlashTx` | evidência de equívoco de âncora + banimento (M9) |
| `0xE2` | `TransferDomain` | transferência de um domínio a um novo owner (M8c) |
| `0x71` | `Heartbeat` | prova de vivacidade de um operador (RFC proof-of-service) |
| `0xA9` | `SetTldOperator` | delegação da operação de um TLD privado (RFC proof-of-service) |
| `0x5E` | `SetDomainOperator` | delegação da operação de um domínio (RFC proof-of-service) |

Qualquer byte de versão diferente de `0x01` é explicitamente rejeitado
(`UnsupportedVersion(valor recebido)`): um formato desconhecido ou obsoleto
nunca é analisado por acidente.

## SLASH_TX (0x15, M9)

Evidência on-chain de que uma âncora assinou dois **checkpoints
conflitantes na mesma época** (equívoco). Qualquer um pode submetê-la — a
evidência é criptográfica e autossuficiente:

```text
offender[32] | evidence_a (CheckpointData 116 B)
  | sig_a[64] | evidence_b (CheckpointData) | sig_b[64]
  | reporter_pk[32] | reporter_signature[64]
```

- `validate()` verifica a estrutura: mesma época, `signing_hash`es
  distintos, duas assinaturas distintas, assinaturas do repórter sobre
  o payload canônico;
- regras de estado (jornalizadas, reversíveis em reorg): AMBAS as
  assinaturas do acusado devem verificar contra os DOIS hashes de
  assinatura distintos na mesma época; o acusado deve estar no pool
  elegível (`SlashOffenderNotInPool` caso contrário); depois
  **banimento** — a chave é removida do pool PoS persistentemente
  (blacklist em estado);
- sanção mínima: os domínios/TLDs do acusado NÃO são confiscados
  (decisão M9 documentada);
- o mesmo checkpoint duas vezes = não é equívoco (rejeitado); épocas
  diferentes = rejeitado; assinatura forjada = rejeitado.

## REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

| Campo | Tipo | Restrições |
|---|---|---|
| `name` | `str` ≤ 253 | nome canônico do domínio reivindicado, **carregado em texto claro** (ver abaixo) |
| `domain_id` | 32 bytes brutos | **deve** igualar `DomainId(name)` (recomputado, nunca confiada) |
| `owner` | 32 bytes brutos | **deve** ser a derivação de `public_key` (ver abaixo) |
| `timestamp` | varint | informação de ordenação (Unix, segundos) |
| `proof` | `bytes` ≤ 256 | payload do PoW de registro (ver [abaixo](#registration-proof-of-work-pow)) |
| `public_key` | 32 bytes brutos | chave pública Ed25519 do assinante (RFC 8032) |
| `signature` | **exatamente 64 bytes** | assinatura Ed25519 do payload assinado |

### O nome é carregado pela transação

`RegisterDomain` transporta o nome canônico em texto claro, seguido
do seu `DomainId`. A consistência `domain_id == DomainId(name)` é
reverificada na codificação, decodificação e validação (defesa em
profundidade): um `RegisterDomain` cujo id não corresponde ao nome
carregado é inválido. O `DomainId` permanece a identidade de consenso
(chaves da chain e do DHT); o nome carregado torna a chain
autodescritiva e cobre toda a reivindicação na assinatura.

## REGISTER_TLD (0x93)

Reivindicação de um domínio de topo (registro de TLDs, M7):

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) open(0x00|0x01) public_key[32] signature[64]
```

| Campo | Tipo | Restrições |
|---|---|---|
| `tld` | `str` ≤ 63 | nome canônico do TLD **carregado em texto claro** (M8b: o desafio PoW deriva do nome, assim como `RegisterDomain` carrega o seu) |
| `tld_id` | 32 bytes brutos | identidade do TLD: `BLAKE3-256("SCONE-TLD-V1" ‖ tld)` — **deve** ser a derivação de `tld` (recomputada) |
| `owner` | 32 bytes brutos | derivação de `public_key` |
| `timestamp` | varint | informação de ordenação (Unix, segundos) |
| `proof` | `bytes` ≤ 256 | payload do PoW de registro (ver [abaixo](#registration-proof-of-work-pow)) |
| `open` | `0x00\|0x01` | nascimento **aberto**: o TLD cai direto em `TldState::Open` (de ninguém, autoatendimento + PoW, um único PoW na tarifa `open_tld_pow_difficulty` — mais barato que privado: recurso comum, sem assento PoS nem exclusividade); nascimento **privado**: comportamento histórico (owner, assento PoS, `tld_pow_difficulty`) |
| `public_key` | 32 bytes brutos | chave pública Ed25519 do assinante |
| `signature` | **exatamente 64 bytes** | assinatura Ed25519 do payload assinado |

O espaço `TldId` é **disjunto** do espaço `DomainId` (prefixos de derivação
distintos — ver `/docs/en/general/naming.md`): registrar o
TLD `uip` (uma reivindicação sobre o namespace `*.uip`) jamais pode ocupar a
identidade de um domínio, nem o contrário. Regra de estado na aplicação: o
TLD deve estar livre (registro separado do registro de domínios).

## A família M8a — formatos wire

Cinco tipos completam o registro de TLDs e o ciclo de vida dos domínios.
Formatos wire normativos desde M8a; **regras de estado completas desde
M8b** (ver `/docs/en/technical/blockchain.md`): owner do TLD exigido, PoW
sempre que uma reivindicação é autoatendimento, expirações e carências do
lado dos domínios. Nenhum payload carrega um campo `network` no wire (ele
vive apenas no payload de assinatura).

### TRANSFER_TLD (0x47)

```text
tld_id[32] owner[32] new_owner[32] public_key[32] signature[64]
```

| Campo | Tipo | Restrições |
|---|---|---|
| `tld_id` | 32 bytes brutos | o TLD transferido |
| `owner` | 32 bytes brutos | derivação de `public_key` (owner atual / assinante) |
| `new_owner` | 32 bytes brutos | identidade do destinatário (opaca: qualquer valor é sintaticamente válido; a ligação às chaves que podem gastá-lo é uma regra de estado M8b) |
| `public_key` | 32 bytes brutos | chave pública Ed25519 do assinante |
| `signature` | **exatamente 64 bytes** | assinatura Ed25519 do payload assinado |

Sem timestamp nem sequência: um TLD tem exatamente um owner por vez; a
primeira transferência aplicada vence (apenas a ordem da chain).

### REVOKE_TLD (0x6B)

```text
tld_id[32] owner[32] public_key[32] signature[64]
```

Abandono voluntário de um TLD (reassumível depois por um
`RegisterTld` fresco). Assinado pelo owner atual; deliberadamente sem
timestamp nem sequência — um revoke é one-shot contra o estado
atual.

### SET_TLD_OPEN (0xB8)

```text
tld_id[32] owner[32] open u8 open_proof(proof) public_key[32] signature[64]
```

| Campo | Tipo | Restrições |
|---|---|---|
| `open` | 1 byte | **estritamente `0x00` ou `0x01`** (forma canônica; qualquer outro valor → erro de decodificação, nunca coerção) |
| `open_proof` | `proof` (length-varint + bytes) | campo **assinado**; obrigatório e verificado quando `open = 0x01` (PoW de abertura, `open_tld_pow_difficulty` da rede, desafio `SCONE-TLD-OPEN-V1 ‖ tld_id`); vazio quando `open = 0x00` (fechar é grátis) |

Um TLD **fechado** = somente assign (domínios criados exclusivamente pelo
owner do TLD via `AssignDomain`); um TLD **aberto** = qualquer um pode fazer
o PoW de registro e reivindicar um domínio livre (`RegisterDomain`).
**Abrir** custa uma prova de trabalho extra (mais leve que a reivindicação
do TLD: ~3 dias na mainnet contra ~1 semana), paga uma vez pelo owner no
`SetTldOpen` — a prova viaja no campo assinado `open_proof` e
não pode ser substituída após a assinatura.

### ASSIGN_DOMAIN (0xD4)

```text
name(str ≤ 253) domain_id[32] owner[32] assignee[32] public_key[32] signature[64]
```

Espelho do `RegisterDomain`: o nome canônico é carregado em
texto claro e `domain_id` deve igualar `DomainId(name)` (recomputado na
decodificação, codificação e validação). Assinado pelo **owner do
TLD do nome** (verificado na aplicação, M8b); `assignee` torna-se o
primeiro owner do domínio (opaco, como `new_owner`).

### RENEW_DOMAIN (0x3C)

```text
name(str ≤ 253) domain_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| Campo | Tipo | Restrições |
|---|---|---|
| `name` | `str` ≤ 253 | nome canônico **carregado em texto claro** (owner-option: a renovação adia a morte do TLD aberto subjacente, a chain precisa conhecer o nome); `domain_id` deve igualar `DomainId(name)` (recomputado) |
| `valid_until` | varint | nova expiração do registro (Unix, segundos) |
| `proof` | bounded | **autorização de renovação** (ver abaixo) |

Assinado pelo owner atual do domínio; se `valid_until` realmente estende
a expiração (e no máximo um mandato de renovação) é uma regra de estado
(M8b).

#### Autorização de renovação — PoW ou PoS conforme o TLD

O custo de uma renovação depende da natureza do TLD subjacente
(avaliado no momento da aplicação, sobre o estado do bloco):

- **TLD aberto**: a renovação custa o **mesmo PoW da criação** —
  mesmo desafio (`SCONE-DOMAIN-V1 ‖ canonical name`), mesma
  dificuldade por comprimento de label e por rede, mesma janela de
  validade de época. Um domínio nunca é "colecionado" de graça: todo mandato
  de vida é pago em trabalho.
- **TLD privado**: o namespace é a aposta (stake) do owner do TLD (PoS).
  A transação deve ser **assinada pelo owner do TLD** (ou seu
  operador delegado) — a assinatura É a autorização, o
  campo `proof` deve estar vazio (20 bytes zero). O owner do TLD
  renova os domínios do seu namespace sob suas próprias regras.

### RENEW_TLD (0x8F)

```text
tld(str ≤ 63) tld_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| Campo | Tipo | Restrições |
|---|---|---|
| `tld` | `str` ≤ 63 | nome canônico do TLD; `tld_id` deve igualar `TldId(tld)` (recomputado) |
| `valid_until` | varint | deve ser EXATAMENTE `now + 3 anos` (mandato fixo — rederivado na aplicação, qualquer outro valor rejeitado) |
| `proof` | bounded | PoW de renovação (ver abaixo) |

Assinado pelo owner **privado** atual do TLD. Regras de aplicação:

- aplica-se apenas a um TLD PRIVADO (um TLD aberto vive através de seus
  domínios — rejeição tipada `TldIsOpen`);
- válido durante o mandato E durante a **carência de 60 dias** que
  segue a expiração; além, o TLD está morto (GC) e a
  rejeição é `TldExpiredFrozen`;
- **PoW igual à criação**: desafio
  `SCONE-TLD-V1 ‖ name ‖ current_expiry(BE64)` (o sufixo anti-reuso
  vincula cada PoW a UM mandato), mesma dificuldade por comprimento
  que a reivindicação inicial;
- a renovação NÃO é congelada durante a carência — é a única
  operação de gestão ainda disponível para um owner expirado.

## UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

| Campo | Tipo | Restrições |
|---|---|---|
| `domain_id` | 32 bytes brutos | domínio alvo |
| `owner` | 32 bytes brutos | derivação de `public_key` |
| `sequence` | varint | **> 0**, deve igualar `current + 1` na aplicação |
| `record_hash` | 32 bytes brutos | compromisso com o conjunto de registros DHT |
| `public_key` | 32 bytes brutos | chave pública Ed25519 do assinante |
| `signature` | **exatamente 64 bytes** | assinatura Ed25519 do payload assinado |

Nota: `UpdateDomain` carrega **nenhum** `timestamp` — a ordenação passa
pelo `sequence`. Ele também não carrega o nome: o domínio já está
registrado, o id basta.

### Limites estritos

- `public_key`: **exatamente 32 bytes brutos**, sem prefixo de comprimento.
  Uma pk que não decodifica como ponto de curva → erro
  `InvalidTransactionKey` na decodificação.
- `signature`: **exatamente 64 bytes brutos**, sem prefixo de comprimento,
  sem margem. Qualquer outro tamanho é um erro fatal de decodificação
  (truncamento rejeitado).
- `proof`: length-varint + bytes, limitado a 256
  (`MAX_PROOF_LEN`), verificado antes da alocação.
- `name`: length-varint + bytes UTF-8, limitado a 253
  (`MAX_NAME_LEN`), revalidado pelas regras de nomenclatura de
  `scone-core`.

## Derivação do owner

Regra dura: **o owner é sempre recomputado, nunca confiada**.

```text
PublicKeyRef = BLAKE3-256("SCONE-PUBKEY-V1" || public_key)
OwnerId      = BLAKE3-256("SCONE-OWNER-V1" || PublicKeyRef)
```

Uma transação cujo `owner` não corresponde à derivação de sua própria
`public_key` é inválida (`InvalidOwner` no lado do core,
`OwnerKeyMismatch` no lado da chain), rejeitada na codificação, decodificação
**e** validação da chain (defesa em profundidade).

## Payload assinado

O payload assinado é a codificação canônica da transação
**sem o campo de assinatura**, prefixado por uma tag de separação de
domínio — **com o `network_id` da chain alvo injetado na cabeça**:

```text
signing_payload(tx, network) = "SCONE-TX-SIG-V1"
                              || canonical_encode(tx_without_signature, network)
```

onde `canonical_encode(tx_without_signature, network)` segue a
estrutura wire acima, discriminante e byte de versão incluídos,
parando antes de `signature`, **com `network` como primeiro campo** — este é o
ÚNICO lugar onde a rede aparece. A chain recomputa o payload
com a SUA própria rede: uma transação assinada para outra rede falha na
verificação de assinatura (`InvalidSignature`); este é o mecanismo anti
replay entre chains.

```text
REGISTER_DOMAIN_no_sig = disc(0x21) version(0x01) network name domain_id[32] owner[32]
                            timestamp.v proof public_key[32]
REGISTER_TLD_no_sig    = disc(0x93) version(0x01) network tld tld_id[32] owner[32]
                            timestamp.v proof public_key[32]
UPDATE_DOMAIN_no_sig   = disc(0x52) version(0x01) network domain_id[32] owner[32]
                            sequence.v record_hash[32] public_key[32]
TRANSFER_TLD_no_sig    = disc(0x47) version(0x01) network tld_id[32] owner[32]
                            new_owner[32] public_key[32]
REVOKE_TLD_no_sig      = disc(0x6B) version(0x01) network tld_id[32] owner[32]
                            public_key[32]
SET_TLD_OPEN_no_sig    = disc(0xB8) version(0x01) network tld_id[32] owner[32]
                            open(0x00|0x01) open_proof(proof) public_key[32]
ASSIGN_DOMAIN_no_sig   = disc(0xD4) version(0x01) network name domain_id[32] owner[32]
                            assignee[32] public_key[32]
RENEW_DOMAIN_no_sig    = disc(0x3C) version(0x01) network name domain_id[32] owner[32]
                            valid_until.v proof public_key[32]
TRANSFER_DOMAIN_no_sig = disc(0xE2) version(0x01) network domain_id[32] owner[32]
                            new_owner[32] public_key[32]
```

Propriedades:

- determinístico e puro: mesma transação ⇒ mesmo payload em todo lugar;
- o prefixo ASCII `SCONE-TX-SIG-V1` (15 bytes,
  hex `53434f4e452d54582d5349472d5631`) impede qualquer reuso da
  assinatura em outro domínio de hash do protocolo;
- todos os campos exceto `signature` são cobertos — **incluindo o nome
  carregado por um `RegisterDomain`**: modificar qualquer um deles
  invalida a assinatura;
- o `network_id` injetado é coberto: a mesma tx não assinada assinada
  para duas redes produz dois payloads distintos (portanto duas
  assinaturas distintas).

## Prova de trabalho de registro (PoW)

O campo `proof` de `RegisterDomain` e `RegisterTld` carrega a
prova de trabalho de registro (módulo `scone-core::pow`).

### Digest (sal de época, anti pré-mineração)

```text
epoch  = block_height / POW_WINDOW        (POW_WINDOW = 144 blocos)
salt   = BLAKE3-256("SCONE-POW-V1-ARGON2" || network || challenge || epoch_le64)
digest = Argon2id(m, t, p = 1; password = nonce_le64, salt = salt[0..16])
```

`network` é o `network_id` canônico (M8b): uma prova minerada para uma
rede não é uma prova em outra.

`SCONE-POW-V1-ARGON2` também fixa os parâmetros Argon2id por rede
(`pow_params`: mainnet m = 64 MiB, t = 2, p = 1; testnet/devnet
m = 8 KiB, t = 1, p = 1): o núcleo Argon2id (RFC 9106) torna o
custo por hash dominado pela memória (resistência a GPU/ASIC), e uma prova
minerada sob um conjunto de parâmetros nunca verifica sob outro.

`challenge` é a entrada de derivação do objeto reivindicado —
`"SCONE-TLD-V1" || tld` para um TLD, `"SCONE-DOMAIN-V1" || name` para um
domínio: exatamente os bytes hasheados para `TldId` / `DomainId`, de modo
que uma prova nunca é reproduzível entre namespaces nem entre tipos de
objeto. `epoch_le64` e `nonce_le64` são as codificações little-endian de
8 bytes da época e do nonce.

A prova é válida quando o `digest` tem pelo menos `difficulty` bits
iniciais zero.

#### Sal de época — anti pré-mineração

Um digest sem sal não se vincula nem ao tempo nem à altura: qualquer um
poderia pré-minerar provas hoje para milhares de nomes, depois registrá-los
em rajada no lançamento da mainnet (squatting) — a prova era autossuficiente
e válida em qualquer bloco que chegasse primeiro. O digest é salgado com a
**época do bloco que APLICA a transação**:

- `epoch = block_height / POW_WINDOW`, `POW_WINDOW = 144` blocos
  (~2.6 h a 65 s/bloco): uma janela de carência para o usuário honesto
  (minerar, depois submeter dentro da janela), re-mineração obrigatória
  além dela;
- a prova DECLARA sua época no wire; na aplicação, a época declarada deve
  cair na **janela de validade** do bloco aplicador — tolerância apenas
  PARA TRÁS (submissões atrasadas são absorvidas, minerar "para o futuro"
  é impossível), janela dimensionada por
  `epoch_validity_window(difficulty)` = `1 << max(0, difficulty −
  16)` épocas (janela alargada: as altas dificuldades da mainnet mineram
  em dias a semanas, mais que uma época; estocar não ganha nada porque a
  janela ≈ o custo de mineração — cada prova de uma rajada
  custa seus `2^difficulty` hashes completos DENTRO da janela). Com
  dificuldade ≤ 16 bits (testnet, nomes longos), a regra de igualdade
  estrita permanece inalterada;
- admissão no mempool (best effort): a altura do bloco futuro é
  desconhecida — o relay verifica contra a época de
  `current height + 1`; a verificação estrita no momento da aplicação
  permanece a autoridade;
- replay determinístico: a prova CARREGA sua época e a janela
  é função pura de (dificuldade, época do bloco) — um bloco
  histórico reverifica identicamente em qualquer nó.

Quebra de compatibilidade pré-mainnet: o antigo formato compacto (12 bytes)
é rejeitado sem compatibilidade retroativa.

### Payload wire (20 bytes fixos)

```text
nonce_le64 || epoch_le64 || difficulty_le32
```

A dificuldade e a época viajam com a prova, mas são
**declaradas pelo produtor e portanto nunca confiadas**: o verificador
compara a dificuldade contra a **constante do protocolo** para o tipo de
registro (um payload cuja dificuldade declarada difere da constante — menor
*ou* maior — é rejeitado, `InvalidProof`), e a época contra a do bloco
aplicador (`ProofEpochMismatch` em caso de divergência). O nonce é então
reverificado na dificuldade constante, digest recomputado do zero.

### Dificuldades (curva **por comprimento**, M8b + calibração medida)

A dificuldade NÃO é um parâmetro plano: é uma função determinística do
**comprimento do label criado** (`scone-core::pow_length` — o TLD para
`RegisterTld`, o label mais à esquerda para `RegisterDomain`), por
rede:

```text
difficulty(len) = apex − round(0.415 × (len − 1)), clamped at the floor
```

| Rede | TLD privado (apex/piso) | TLD nascido aberto / domínio (apex/piso) |
|---|---|---|
| testnet | 8 bits / 2 bits | 6 bits / 2 bits |
| mainnet | 42 bits / 30 bits | 40 bits / 30 bits |
| devnet | 2 bits / 1 bit | 1 bit / 1 bit |

Um TLD **nascido aberto** (`RegisterTld` com `open = 0x01`) paga a
curva de "abertura" (apex − 2, como `domain_difficulty`); **abrir um
TLD privado existente** (`SetTldOpen`) permanece na dificuldade plana
(`open_tld_pow_difficulty`: 6 bits testnet / 22 bits mainnet /
2 bits devnet — a tx carrega apenas o `tld_id`, o nome pagou seu
comprimento na criação).
Benchmarks da mainnet (TLD privado, ~6 M hashes/s): 1 caractere = 42
bits (≥ 7 dias), 5 caracteres = 40 bits (~2 dias), 19 caracteres = 35 bits
(~1.6 h), 29+ caracteres = piso de 30 bits (~3 min). As curvas vivem em
`scone-core::pow_length` (constantes `MAINNET_APEX = 42`,
`MAINNET_FLOOR = 30`, `TESTNET_APEX = 8`, `TESTNET_FLOOR = 2`,
`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1`); os
parâmetros planos (`tld_pow_difficulty`, etc.) permanecem em
`scone-core::network::NetworkParams`. Mudar um valor = mudança de consenso
**para essa rede**. Inváriante: a devnet é uma terceira
`NetworkParams` completa e a testnet permanece estritamente idêntica às
suas constantes pré-devnet (parâmetros Argon2id, curvas, janelas) — a
devnet NÃO compartilha nenhuma constante com as outras redes além do
código em si.

Calibração medida dos picos planos (2026-10, `cargo run
--release -p scone-core --example pow_bench` — throughput
single-thread ~7.7 M H/s num x86-64 de referência):

| Tipo (mainnet) | dificuldade | tempo real médio |
|---|---|---|
| `RegisterTld` | 24 bits | ~7 s |
| `RegisterDomain` | 20 bits | ~54 ms |

A ~65 s/bloco: uma reivindicação de TLD custa ~0.1 bloco de CPU
single-core, um domínio ~1 ms de bloco — simbólico para um usuário, caro
de industrializar em escala (anti-squatting). Recalibrar se os alvos de
hardware caírem (ARM móvel: ~4–8× mais lento).

Ramos de estado (M8b, entregues): `RegisterTld` SEMPRE exige seu PoW;
`RegisterDomain` o exige num TLD **aberto** (o caminho fechado é
assign-only — `AssignDomain`, sem PoW, assinado pelo owner do TLD).

## Regras de validação

Uma transação é válida se e somente se:

1. **decodificação estrita**: estrutura e limites acima respeitados
   (discriminante conhecido, versão `0x01`, pk 32 B, assinatura 64 B,
   invariantes do core reverificados na decodificação);
2. **vínculo owner/pk**: `owner == OwnerId::from(public_key)`
   (recomputado);
3. **consistência nome/id** (RegisterDomain): `domain_id ==
   DomainId(name)` (recomputado);
4. **assinatura**: `verify_strict` Ed25519 da `signature` contra o
   payload assinado **recomputado do zero** com a rede da chain
   (nunca tomado de um par nem da transação como fornecida) — uma
   tx assinada para outra rede falha aqui (`InvalidSignature`,
   anti replay entre chains);
5. **regras de estado** (na aplicação num bloco — M8b, ver
   `/docs/en/technical/blockchain.md`):
   RegisterTld → TLD livre + não ICANN + PoW (dificuldade
   por comprimento); RegisterDomain → TLD registrado **e aberto** + domínio
   livre (carência incluída) + PoW (por comprimento);
   UpdateDomain → domínio existente, `owner` atual,
   `sequence == current + 1`;
   TransferTld / RevokeTld / SetTldOpen / AssignDomain → TLD existente
   **e privado**, assinante = owner do TLD;
   TransferDomain → domínio existente, owner;
   RenewDomain → domínio existente, extensão estrita ≤ 3 anos,
   autorização PoW (TLD aberto: mesmo PoW da criação,
   **vinculado à expiração atual** — um desafio distinto do
   registro e de toda renovação anterior) ou PoS
   (TLD privado: assinatura do owner do TLD — ou do seu operador efetivo —,
   `proof` vazio).

A verificação usa `verify_strict` (rejeição de assinaturas maleáveis
e chaves fracas/não canônicas), como exigido para dados não confiáveis.

## Ordem das operações em `push_block`

A validação criptográfica (2 a 4) ocorre para cada transação
**antes** da aplicação de estado, sobre um estado de trabalho atômico por
bloco; qualquer erro deixa a chain rigorosamente inalterada.

## Montagem do bloco

`BlockBuilder` (`scone-blockchain`) monta um bloco a partir de uma
fila de transações: `tx_root` recomputado sobre a lista ordenada, header
encadeado ao hash pai (`prev_hash`), `version` =
`PROTOCOL_VERSION`. O payload de consenso (payload de produtor assinado)
e o timestamp são fornecidos pelo chamador; o trait `Consensus` existente
é usado simetricamente com a validação.

## Exemplos hex (vetores reais, reproduzíveis — M8b)

### RegisterDomain `example.uip`, timestamp 42, testnet

Payload assinado (vetor fixado; historicamente gerado pela
ferramenta offline removida — os valores permanecem vetores válidos). A
`network` aparece aqui porque este é o PAYLOAD DE ASSINATURA (seu único
lugar); a transação wire assinada em si não contém nenhuma:

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

### RegisterTld `uip`, timestamp 42

Payload assinado (vetor fixado) — o `TldId` é o vetor de derivação
fixado em `/docs/en/general/naming.md`:

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

(Os valores completos eram reproduzíveis pela ferramenta offline
removida; os vetores abaixo permanecem fixados e válidos. O PoW é
minerado automaticamente na dificuldade da rede — nonces variam de execução
em execução; apenas network/name/id/owner/pk são estáveis. Desde a
mudança de wire de 2026-09, o hex da transação assinada NÃO CONTÉM mais os
bytes `0d 73636f6e652d746573746e6574`: a rede viaja apenas no
payload a assinar.)

## Onde o hex vive agora

Os comandos offline `tx build|sign|verify` foram removidos
(cli-simplify-v1). O formato hex assinado em si permanece inalterado e
continua o formato wire do **fluxo web/API**: o aplicativo web do
registrador constrói, assina (keystore no navegador) e submete
transações como hex canônico pela API. No lado da CLI,
os caminhos do usuário são as formas de um comando:

```bash
scone domain register example.uip        # minera, assina, submete
scone domain renew example.uip                      # prova de renovação, assina, submete
scone domains                            # o que você possui
```

Em todos os caminhos de código o owner e a chave são **recomputados** da
identidade assinante, nunca tomados de qualquer entrada.

## Histórico de versões

| Versão | Conteúdo | Estado |
|---|---|---|
| 1 | `public_key` + `signature` embutidos, payload `SCONE-TX-SIG-V1`, byte de versão `0x01`, `RegisterDomain` carrega o nome, `RegisterTld` (0x93), família M8a: `TransferTld` (0x47), `RevokeTld` (0x6B), `SetTldOpen` (0xB8), `AssignDomain` (0xD4), `RenewDomain` (0x3C) — formatos wire normativos; M8b: campo `network` assinado na cabeça de todo payload, `RegisterTld` carrega o nome, regras de estado completas, dificuldades PoW por rede | substituída |
| 1 (2026-09, wire sem network) | **remoção do campo `network` do wire de todas as transações**: o network_id vive apenas no payload de assinatura (`signing_payload(tx, network)` o injeta na cabeça); a separação de redes apoia-se na assinatura (`InvalidSignature` num payload estrangeiro, o erro `WrongNetwork` desaparece); `TX_FORMAT_VERSION` permanece `0x01` (testnet relançável, sem coexistência); **todos os TxIds mudam** (o TxId é o hash da codificação wire assinada, que perde 14–15 bytes); digest PoW, gênese e data-dirs por rede inalterados | substituída |
| 1 (2026-10, sal de época PoW) | **digest PoW anti pré-mineração**: sal `BLAKE3("SCONE-POW-V1-ARGON2" ‖ network ‖ challenge ‖ epoch_le)` + núcleo Argon2id sobre `nonce_le`, com `epoch = block_height / 144` do bloco que APLICA a tx; payload wire da prova 12 → **20 bytes** (`nonce_le ‖ epoch_le ‖ difficulty_le`), a época declarada é verificada contra a janela de validade de época do bloco aplicador (rejeição tipada `ProofEpochMismatch`), best-effort `(height+1)/144` na admissão do mempool, flag CLI `--epoch` nos caminhos offline; quebra de compatibilidade pré-mainnet: o antigo payload de 12 bytes é rejeitado, **todos os TxIds mudam** (o campo `proof` cresce 8 bytes) | atual |
