# Blockchain

Documento de referência da camada blockchain: identidades, hashing, gênese,
estado canônico, ordenação de transações e validação de blocos.
A implementação de referência é a crate
[`scone-blockchain`](../../../crates/scone-blockchain), sobre
`scone-protocol` (codificação canônica), `scone-core` (regras de negócio) e
`scone-crypto` (BLAKE3).

Escopo atual: lógica blockchain com consenso real (produtor assinado,
checkpoints, finalidade), **estado em memória** sobre um backend KV
(armazenamento concreto vive em `scone-storage`) — ver
[Adiado ao consenso](#adiado-ao-consenso).

## Identidades e separação de domínios de hash

Quatro identidades, quatro papéis, quatro prefixos BLAKE3 distintos — um prefixo
nunca é reutilizado entre dois usos:

| Tipo | Papel | Prefixo | Fórmula |
|---|---|---|---|
| `DomainId` | identidade do nome de domínio | `SCONE-DOMAIN-V1` | ver `/docs/en/general/naming.md` |
| `RecordHash` | identidade do conteúdo DNS | `SCONE-RECORD-V1` | ver `/docs/en/technical/protocol.md` |
| `TxId` | identidade da transação | `SCONE-TX-V1` | abaixo |
| `BlockHash` | identidade do bloco | `SCONE-BLOCK-V1` | abaixo |

A árvore Merkle usa seu próprio prefixo `SCONE-MERKLE-V1`. Estes prefixos
somam-se aos de `scone-core` (`SCONE-PUBKEY-V1`,
`SCONE-OWNER-V1`).

Cadeia de dependências completa:

```text
DomainId
    ↓
RegisterDomain / UpdateDomain
    ↓
Transaction
    ↓
TxId
    ↓
MerkleRoot
    ↓
BlockHeader
    ↓
BlockHash
    ↓
prev_hash do próximo bloco
    ↓
chain canônica
```

## TxId

```text
TxId = BLAKE3-256("SCONE-TX-V1" || canonical_encode(Transaction))
```

- determinístico: a mesma transação (mesma codificação canônica) ⇒ mesmo
  `TxId`, em todos os nós, para sempre;
- identidade **por conteúdo**: qualquer campo modificado muda o `TxId`
  (domínio, owner, timestamp, proof, sequence, record_hash);
- nenhum timestamp local ou valor não canônico entra na computação;
- nenhuma dependência de armazenamento ou rede;
- a identidade não garante unicidade de aplicação: reproduzir uma
  transação idêntica falha nas regras de estado (duplo `RegisterDomain`,
  replay de sequência), não no `TxId`.

API: `scone_blockchain::transaction_id(&Transaction) -> Result<TxId>`.

## Árvore Merkle (tx_root)

Compromisso sobre a **lista ordenada** das transações de um bloco:

```text
0 transactions  root = BLAKE3-256("SCONE-MERKLE-V1" || "EMPTY")
leaf            BLAKE3-256("SCONE-MERKLE-V1" || 0x00 || TxId)
internal node   BLAKE3-256("SCONE-MERKLE-V1" || 0x01 || left || right)
odd level       the last hash is duplicated to complete the pair
```

Regras:

- **nunca ordenada**: a ordem das transações num bloco é
  significativa; dois blocos contendo as mesmas transações em
  ordem diferente têm raízes diferentes;
- as tags `0x00` / `0x01` impedem qualquer confusão entre uma folha e um
  nó interno (segunda pré-imagem);
- o prefixo `SCONE-MERKLE-V1` isola a árvore dos outros domínios de
  hash (um `TxId` não pode ser interpretado como um nó);
- implementação própria (~30 linhas), determinística e testada; nenhuma
  biblioteca Merkle externa.

API: `merkle_root(&[TxId]) -> MerkleRoot` e
`tx_root(&[Transaction]) -> Result<MerkleRoot>`.

## BlockHash

```text
BlockHash = BLAKE3-256("SCONE-BLOCK-V1" || canonical_encode(BlockHeader))
```

O hash cobre **apenas** o header canônico: as transações já
estão comprometidas pelo `tx_root`, portanto não são hasheadas uma segunda
vez:

```text
transactions -> TxIds -> MerkleRoot -> BlockHeader -> BlockHash
```

Modificar uma transação, sua ordem, a altura, o timestamp, o
`prev_hash` ou o payload de consenso muda o `BlockHash`.

API: `block_hash(&BlockHeader) -> Result<BlockHash>`.

## Redes e gênese (M8b)

O Scone define três redes disjuntas, identificadas por um `network_id`
canônico (1..=16 bytes ASCII `[a-z0-9-]`):

| Rede | `network_id` | apex TLD privado | apex TLD aberto | apex domínio | piso TLD | piso domínio |
|---|---|---|---|---|---|---|
| testnet | `scone-testnet` | 8 bits | 7 bits | 6 bits | 2 bits | 1 bit |
| mainnet | `scone-mainnet` | 28 bits | 27 bits | 26 bits | 25 bits | 22 bits |
| devnet | `scone-devnet` | 2 bits | 1 bit | 1 bit | 1 bit | 1 bit |

Os parâmetros vivem em `scone-core::network::NetworkParams`
(instâncias nomeadas `TESTNET` / `MAINNET` / `DEVNET`, consulta via
`NetworkParams::by_name`). Mudar uma dificuldade = mudança de consenso
**para essa rede**.

**Inváriante estrutural**: a devnet é uma terceira
`NetworkParams` completa — nunca um subconjunto de flags da testnet. A
testnet permanece **estritamente idêntica** ao que era antes de a devnet
ser adicionada (mesmas constantes Argon2id, mesmas curvas, mesmas janelas
de época); a suíte de testes fixa este inváriante.

**Dificuldade variável por comprimento** (`scone-core::pow_length`): a
dificuldade de uma criação depende do comprimento do **label criado**
(o TLD para `RegisterTld` — privado ou recém-aberto —, o label mais à
esquerda para `RegisterDomain`). A fórmula dá:

```text
TLD     : difficulty(len) = apex − round(0.287 × (len − 1)), clamped at the floor
domain  : difficulty(len) = apex − round(0.480 × (len − 1)), clamped at the floor
```

Duas inclinações distintas: `log₂(1.22) ≈ 0,287` bit/caractere para TLDs
(cada caractere extra divide o custo por ~1,22), e
`log₂(1.40) ≈ 0,480` bit/caractere para domínios — domínios decaem
mais rápido: seu valor está no TLD que os carrega. Um TLD nascido aberto
fica no apex privado − 1; um domínio no apex privado − 2
(mainnet: TLD privado 25, TLD aberto 24, domínio 23). Benchmarks da
mainnet (Argon2id memory-hard, ~11.4 H/s single-thread):
TLD privado 1 caractere = 28 bits ≈ 30 dias, 9+ caracteres = piso de 25
bits ≈ 3.8 dias; TLD aberto 1 caractere = 27 bits ≈ 15 dias; domínio
1 caractere = 26 bits ≈ 3.8 dias, 9+ caracteres = piso de 22 bits
≈ 11.5 h. **Máquina de referência de 8 núcleos**
(recalibração 2026-10 — a CLI minera
multi-core), ~11.4 H/s/núcleo. **Abrir** um
TLD privado existente (`SetTldOpen`) permanece na dificuldade plana
(`open_tld_pow_difficulty`: 6 bits testnet / 22 bits mainnet): a tx
carrega apenas o `tld_id`, o nome pagou seu comprimento na criação.
Função pura (determinismo: dois nós computam a mesma dificuldade
para o mesmo nome).

A gênese é uma constante de protocolo **por rede**, reproduzível a partir
das constantes sozinhas (nenhuma aleatoriedade, nenhum relógio local,
nenhuma rede):

| Campo | Valor |
|---|---|
| `version` | `PROTOCOL_VERSION` (`1`) |
| `height` | `0` |
| `prev_hash` | 32 bytes zero (nenhum bloco anterior) |
| `tx_root` | raiz Merkle da lista vazia |
| `timestamp` | `GENESIS_TIMESTAMP = 0` (puramente estrutural) |
| `consensus` | os bytes canônicos do `network_id` |
| transações | vazio |

Variante de **gênese de alocação** (kickstarter da mainnet, M-next): o
campo `consensus` carrega `network_id ‖ ALLOC-V1 ‖ count.v ‖ entries`
(domínios/TLDs pré-pagos, ordenação canônica por nome, manifesto ≤ 60
KiB, ≤ 2048 alocações) — este formato alargado é o que justifica
`MAX_CONSENSUS_LEN` = 64 KiB (ver `scone-blockchain::genesis_alloc`).

O `network_id` no payload de consenso torna os hashes de gênese —
e portanto toda a blockchain — **disjuntos entre redes**: um
bloco da testnet jamais pode se prender a uma chain da mainnet
(`UnknownParent`), nenhum replay é possível sem reassinar
tudo.

API: `genesis_of(&NetworkParams) -> Block`,
`genesis_hash_of(NetworkId) -> BlockHash`; os aliases `genesis()` /
`genesis_hash()` referem-se à testnet (padrão de desenvolvimento).

### Devnet — rede local reiniciável

A devnet (`scone-devnet`) é a terceira rede, dedicada à iteração local
rápida:

- **Gênese própria**: o id de rede `scone-devnet` no payload de consenso
  dá um hash de gênese distinto da testnet E da
  mainnet — nenhuma gênese, bloco, assinatura ou prova PoW jamais cruza a
  fronteira (replay impossível).
- **PoW quase zero**: curva fixada na faixa 0..=2 bits
  (`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1` — o decaimento por comprimento
  continua a valer): uma reivindicação de TLD ou domínio minera em
  milissegundos, exercitando exatamente o mesmo caminho de código que as
  outras redes.
- **Consenso PoS rápido**: mesmos parâmetros que a testnet (comitê de
  4, épocas de 10 s) para finalidade rápida numa malha local.
- **Reset**: a chain vive em seu próprio diretório de dados
  (`~/.scone/devnet/` por padrão). **Apagar esse diretório e
  relançar o relay reseta a chain** (gênese fresca, store virgem)
  — nenhuma ferramenta extra necessária. NUNCA apontar um data dir de
  devnet para o data dir de outra rede: um store não vazio contendo outra
  chain é recusado (`Corrupted`).

## Estado canônico

```text
DomainState  = { owner: OwnerId, sequence: u64, record_hash: Option<RecordHash>,
                 registered_at: u64, valid_until: u64 }
TldState     = enum { Private { owner: OwnerId } | Open }
               -- enum (owner-option): o inváriante aberto/privado é
               -- estrutural; o prazo de morte vive fora da entrada
               -- (tabela tld_floors + agenda de morte jornalizada)
ChainState   = NetworkParams                  (rede vinculada, M8b)
               + DomainId -> DomainState      (acesso direto, em memória)
               + TldId -> TldState            (registro de TLDs, M7b)
               + DomainId -> (expired_at, OwnerId)   (janelas de carência, M8b)
```

* `registered_at` / `valid_until` (M8b): expiração do registro,
  avaliada contra o **timestamp do bloco pai** (determinístico,
  comprometido no header — nunca o relógio local);
* `open`: um TLD privado pode **nascer aberto** (`RegisterTld` com
  `open = 0x01` — de ninguém, autoatendimento) ou **fechado** na
  reivindicação (fechado = somente assign), depois tornar-se aberto via
  `SetTldOpen` (auto-registro com PoW) — **irreversivelmente**;

- acesso direto por `DomainId` (32 bytes): nunca uma `String` como chave,
  nunca uma varredura completa — condição necessária para mirar centenas de
  bilhões de domínios;
- `record_hash == None` enquanto nenhum `UpdateDomain` foi aplicado;
- os espaços de ids de TLD e domínio são **disjuntos por construção**
  (prefixos de derivação distintos, ver `/docs/en/general/naming.md`):
  registrar o TLD `uip` jamais pode ocupar a identidade de um domínio, nem
  o contrário.

### Regras de aplicação

`state.apply(transaction)` é determinístico e atômico (em erro, o
estado fica inalterado):

**Todas as transações (M8b, wire 2026-09)**: as transações não carregam
mais um campo `network` — a separação apoia-se na **assinatura**: o
payload assinado injeta o `network_id` da chain
(`validate_transaction(tx, network_of_the_chain)`), de modo que uma tx
assinada para outra rede falha na verificação de assinatura
(`InvalidSignature`, ANTES da aplicação — uma tx assinada na testnet
nunca é aplicada, nunca entra no mempool de um nó da mainnet, e
vice-versa).

**REGISTER_TLD**:

- o TLD deve estar **livre** (ausente do registro de TLDs) e **não
  reservado ao ICANN** (`IcannTldReserved`: os TLDs raiz do ICANN não
  pertencem a ninguém no Scone);
- **PoW obrigatório**: `pow::verify` sobre o desafio
  `SCONE-TLD-V1 || tld` (derivado do nome carregado pela tx), na
  dificuldade **por comprimento** do label (`pow_length`):
  nascimento privado → `tld_difficulty` (apex da rede), nascimento aberto
  (`open = 0x01`) → `open_tld_difficulty` (apex − 2) — o digest é
  recomputado do zero
  (sal `SCONE-POW-V1-ARGON2 ‖ network ‖ challenge ‖ epoch`,
  núcleo Argon2id), a
  dificuldade autodeclarada deve igualar a dificuldade esperada, e a
  **época declarada** pela prova deve igualar
  `height/POW_WINDOW` do bloco aplicador (sal de época contra
  pré-mineração — rejeição tipada `ProofEpochMismatch`, ver
  `transactions.md`);
- na aplicação: `open = 0x01` → `{ Open }` (de ninguém — autoatendimento,
  assento PoS liberado, piso de morte lançado em `now + grace`);
  `open = 0x00` → `{ Private { owner } }` — **fechado por padrão**
  (somente assign); a abertura é explícita via `SetTldOpen`.

**REGISTER (RegisterDomain)**:

- **o TLD do nome carregado deve estar registrado E aberto**: a
  chain deriva `TldId(name.tld())` ela mesma (nunca um valor
  fornecido); ausente → `UnknownTld` (D1, M7c), presente mas fechado →
  `TldClosed` (M8b: o caminho somente-assign é exclusivo);
- o domínio deve estar **livre**: ausente do estado E fora de qualquer
  janela de carência de um registro expirado (ver GC abaixo);
- **PoW obrigatório**: desafio `SCONE-DOMAIN-V1 || name` na
  **dificuldade por comprimento do label mais à esquerda**
  (`pow_length::domain_difficulty` — o TLD sendo aberto, este é o
  preço de entrada em autoatendimento);
- na aplicação: `{ owner, sequence: 0, record_hash: None,
  registered_at: now, valid_until: now + 1 ano }` — **o registro
  dura 1 ano** (365 dias).

**Ordem intra-bloco**: significativa. `[RegisterTld, SetTldOpen,
RegisterDomain]` no mesmo bloco é válido; qualquer permutação que
aplique uma regra antes de sua pré-condição existir é rejeitada como um
bloco inteiro (undo-log).

**UPDATE (UpdateDomain)**:

- o domínio deve existir (não expirado);
- `tx.owner` deve ser o owner atual;
- `tx.sequence` deve ser **exatamente** `current_sequence + 1`
  (sem lacunas, sem replay);
- na aplicação: `sequence` avança, `record_hash` é substituído.

**TRANSFER_TLD (0x47)**:

- o TLD deve existir e ser **privado** (`UnknownTld` / `TldIsOpen`
  caso contrário — um TLD aberto não pertence a ninguém);
- o assinante deve ser o owner atual (`NotTldOwner` caso contrário);
- na aplicação: `owner = new_owner` (a identidade do destinatário é
  opaca; a ligação às chaves que podem gastá-lo é feita por
  txs subsequentes). Sem sequência: a primeira transferência aplicada
  vence (ordem da chain).

**REVOKE_TLD (0x6B)**:

- o TLD deve existir e ser **privado** (um TLD aberto não pode ser
  revogado — `TldIsOpen`: ele morre naturalmente no GC), assinante =
  owner atual;
- na aplicação: o TLD é removido do registro e torna-se
  reassumível por um `RegisterTld` fresco (PoW incluído).

**SET_TLD_OPEN (0xB8)**:

- o TLD deve existir e ser **privado** (já aberto → `TldIsOpen`:
  o antigo owner é apagado, não existe mais chave legítima para
  assinar por ele); `open = 0x00` (fechar) → rejeição tipada
  `TldCloseUnsupported` já na validação pura: **a abertura é
  irreversível** (owner-option);
- assinante = owner atual (`NotTldOwner`);
- **abertura = PoW obrigatório**: `pow::verify` sobre o desafio
  `SCONE-TLD-OPEN-V1 || tld_id` na dificuldade plana da
  rede `open_tld_pow_difficulty` (6 bits testnet / 22 bits mainnet),
  sal de época incluído (a prova viaja no campo assinado
  `open_proof` da tx);
- na aplicação: `TldState::Open` — **o owner é apagado**
  (irreversível, assento PoS liberado), um piso de morte é lançado em
  `now + grace`.

**ASSIGN_DOMAIN (0xD4)**:

- o TLD do nome carregado deve existir e ser **privado**
  (`TldIsOpen` caso contrário: um TLD aberto é somente autoatendimento),
  assinante =
  owner do TLD (`NotTldOwner`);
- o domínio deve estar livre (mesma regra de carência que REGISTER);
- na aplicação: `{ owner: assignee, sequence: 0, record_hash:
  None, registered_at: now, valid_until: now + 1 ano }`. **Sem
  PoW**: é o owner do namespace quem garante.

**TRANSFER_DOMAIN (0xE2, M8c)**:

- o domínio deve existir (`UnknownDomain` caso contrário), assinante =
  owner atual (`NotOwner`);
- na aplicação: `owner = new_owner` (o assento PoS segue o
  domínio no pool de owners, simétrico ao `TransferTld`). Sem
  sequência: a primeira transferência aplicada vence (ordem da chain).

**RENEW_DOMAIN (0x3C)**:

- o domínio deve existir; **o tipo de TLD é avaliado PRIMEIRO**
  (endurecimento M8c):
  - **TLD privado**: autorização **PoS** — o assinante É o
    owner do TLD, ou o assento do owner do TLD delega para a chave
    pública do assinante
    (`effective_operator(tld_owner) == renew.public_key` — a
    delegação é sempre lida A PARTIR DO owner do TLD, nunca o
    contrário). O campo `proof` deve estar vazio (proof não vazio
    = rejeitar).
  - **TLD aberto**: autorização **PoW** — o assinante é o
    owner atual do domínio (`NotOwner` caso contrário) e a prova
    mina o desafio de renovação: mesma dificuldade que a criação
    (por comprimento do label), **vinculada à expiração atual**
    (`challenge = DOMAIN_ID_VERSION ‖ canonical name ‖ current
    valid_until`) — a prova de uma renovação é distinta da do
    registro e de toda renovação anterior
    (anti-reuso);
- `valid_until` deve **estritamente estender** o prazo atual
  (`RenewalNotExtending` caso contrário);
- `valid_until ≤ now + 3 anos` (`RenewalExceedsTerm` —
  limite anti-acúmulo: **1 mandato de renovação = 1 ano, no máximo 3
  anos de antecedência**);
- na aplicação: `valid_until = tx.valid_until`; a renovação também
  adia o piso de morte do TLD aberto subjacente (a tx
  carrega o nome canônico para esse fim).

**GC determinístico (M8b)**: antes de aplicar as transações de um
bloco, a chain avalia as expirações no **timestamp do bloco pai**:
qualquer domínio cujo `valid_until ≤ now` é removido do estado vivo
e estacionado numa **janela de carência de 30 dias**
(`valid_until + 30 dias`). Durante a carência, apenas o antigo
owner pode reassumir o nome; depois, ele fica livre para todos.
O GC é jornalizado: um bloco rejeitado restaura o estado pré-GC bit a
bit.

**Ciclo de vida dos TLDs (mandatos fixos)**: os TLDs carregam seu
próprio relógio, independente dos domínios.

- **TLD privado**: a reivindicação abre um **mandato de 3 anos**
  (`TLD_TERM_SECS`). A renovação (`RenewTld`, 0x8F) paga um **PoW igual
  à criação** e reseta exatamente 3 anos (mandato fixo
  rederivado na aplicação). Na expiração, uma **carência de 60 dias**
  (`TLD_GRACE_SECS`) abre: a renovação permanece possível, mas **toda
  outra operação de gestão é congelada** — notadamente `AssignDomain`
  (rejeição tipada `TldExpiredFrozen`): um owner expirado perde
  direitos de gestão até renovar. Domínios já atribuídos abaixo
  vivem seu próprio `valid_until` — o TLD congela, não seus
  filhos. Carência esgotada sem renovação: o TLD morre no GC
  (assento PoS liberado, reassumível). Um `TransferTld` carrega o
  mandato restante.
- **TLD aberto**: nasce (ou é aberto) com um **piso de 1 ano**
  (`OPEN_TLD_FLOOR_SECS` — anti criar-destruir instantâneo). Cada
  domínio vivo abaixo empurra o relógio da morte para trás (ele segue
  a expiração do domínio mais recente). Nenhum domínio restante +
  piso esgotado: morte no GC, reassumível.
- **TLD aberto imortal** (apenas gênese: `.scone`, `.tor` —
  nomenclatura verificável sobre endereços onion via CNAME —, `.p2p`): a
  alocação `OpenImmortalTld` põe o sentinela `until = u64::MAX`
  — o GC jamais pode cruzá-lo e o rearme monotônico
  jamais pode rebaixá-lo: o TLD nunca expira, mesmo esvaziado de
  todos os seus domínios. Nenhuma transação em runtime pode criar um
  imortal (`RegisterTld` mantém seu piso de 1 ano); os domínios
  abaixo vivem seu ciclo normal.

As mortes de TLD compartilham a fila de morte determinística dos
domínios (`tld_deaths`), jornalizada e restaurável bit a bit.

**Exceção da altura-1**: a gênese carrega um timestamp estruturalmente
nulo (`GENESIS_TIMESTAMP` = 0, convenção de protocolo — não um
relógio). O tempo da chain do PRIMEIRO bloco é portanto seu próprio
timestamp (comprometido em seu header, determinístico), não o do pai:
ancorar as expirações do bloco 1 em 0 prenderia o piso de morte de um
TLD nascido aberto (e o `valid_until` de um domínio nascido no bloco 1)
à época Unix — o GC pré-aplicação do bloco 2 (timestamp real) removeria o
recurso antes da aplicação ("dropped conflicting tx from block
production: unknown TLD", regressão testnet 60).

**Replay**: o replay entre redes é impossível em três níveis —
gêneses disjuntas (`UnknownParent`), `network_id` injetado em
todo payload de assinatura (assinatura inválida na outra rede —
o único lugar onde a rede aparece desde o wire 2026-09), PoW separado
por rede no digest. O replay intra-rede de uma tx idêntica
falha nas regras de estado (dupla reivindicação, sequência), nunca no
`TxId`.

Dois nós da mesma rede partindo do mesmo estado e
aplicando os mesmos blocos na mesma ordem produzem exatamente o mesmo
estado final.

## Ordenação das transações

A ordem é uma propriedade fundamental, comprometida em todos os níveis:

```text
ordem das transações no bloco
    ↓
MerkleRoot
    ↓
BlockHash
    ↓
chain canônica
```

Duas transações não são intercambiáveis: dentro do mesmo bloco,
`[REGISTER, UPDATE]` aplica-se, `[UPDATE, REGISTER]` é rejeitado
(domínio inexistente no momento do `UPDATE`). O desempate global (que
ordena o mempool, que produz o bloco) pertence ao consenso.

## Validação de blocos (push_block)

Tudo é **recomputado**, nada é aceito na fé — um `tx_root`,
hash, assinatura ou campo fornecido por um par nunca é aceito ao pé da
letra:

1. `prev_hash` == hash do tip canônico (caso contrário `UnknownParent`
   se o pai é desconhecido, `ParentNotTip` se é conhecido mas não é o
   tip: detecção mínima de fork);
2. `height` == altura do tip + 1;
3. `version` == `PROTOCOL_VERSION` **exatamente** (qualquer outra
   versão — menor ou maior — é rejeitada, não reinterpretada);
4. contagem de transações ≤ `MAX_TXS_PER_BLOCK` (4096);
5. `tx_root` recomputado sobre as transações na ordem do bloco;
6. hooks de consenso (`validate_header`, `validate_tx`);
6ter. **anti-replay TXID em janela (porta .bak)**: uma transação cujo
   `TxId` aparece no índice de inclusão dos
   `REPLAY_WINDOW_BLOCKS = 256` blocos mais recentes é rejeitada
   (`TxReplay`). O índice (TxId → altura de inclusão) é função pura
   da chain canônica, podado no mesmo ritmo; o
   índice é reconstruído no boot pelo storage (rescan dos últimos
   `min(256, height)` blocos persistidos — M8, ver
   `/docs/en/technical/storage.md`): um nó reiniciado rejeita os mesmos
   replays que um nó vivo. Além da janela, a reinclusão permanece
   sujeita às regras de estado (dupla reivindicação, replay de
   sequência).
6bis. **produtor assinado (M5, porta .bak)**: o payload `consensus`
   deve decodificar como um payload de produtor
   (`SCONE-BLOCK-V1 ‖ pk[32] ‖ sig[64]`, limitado); a assinatura
   cobre o *hash de assinatura do produtor* = o `BlockHash` do header
   **com o campo `consensus` esvaziado** (o payload vive nesse
   campo; assinar o hash completo seria circular); o produtor deve
   pertencer ao conjunto autorizado no `timestamp` do bloco (comitê
   eleito por checkpoint, mais sorteios de recuperação destravados;
   bootstrap: owners de domínios vivos — um pool vazio = produção
   aberta, caso contrário o primeiro registro seria impossível). A
   gênese (altura 0, payload vazio) é estrutural e nunca concernida.
   Erro tipado `InvalidProducer`;
7. para cada transação, em ordem: validação **criptográfica**
   (`validate_transaction`: vínculo owner/chave recomputado +
   `verify_strict` sobre o payload assinado recomputado do zero, ver
   `/docs/en/technical/transactions.md`), depois aplicação ao
   estado canônico sob um **diário de desfazer** (undo-log: uma
   entrada por transação aplicada, custo O(transações do bloco), não
   O(domínios));
8. commit atômico: tudo passa ⇒ bloco adicionado, diário descartado;
   o menor erro ⇒ rebobinamento do diário (ordem inversa) e a chain
   rigorosamente inalterada, bit a bit.

Nenhuma função de validação entra em pânico; qualquer entrada malformada
ou hostil produz um `BlockchainError` tipado
(`OwnerKeyMismatch`, `InvalidSignature`, …).

## Montagem de blocos (BlockBuilder)

`BlockBuilder` (`scone-blockchain::builder`) monta um bloco a partir de
uma fila de transações validadas:

```text
BlockBuilder::after(parent_height, prev_hash)
    .with_timestamp(t) .with_consensus(payload)
    .push_tx(tx)… .build() -> Block
```

- `tx_root` **recomputado** sobre a lista ordenada das transações em
  fila (nunca fornecido);
- `prev_hash`/`height` encadeados ao pai (o hash do pai é
  computado pelo chamador via `block_hash`, nunca tomado de um par);
- `version` = `PROTOCOL_VERSION`;
- o limite `MAX_TXS_PER_BLOCK` imposto na inserção
  (`TooManyTransactions`);
- o payload `consensus` (payload de produtor assinado) e o `timestamp`
  são
  fornecidos pelo chamador: o consenso concreto pluga via trait
  [`Consensus`](#abstração-de-consenso) existente, simetricamente à
  validação;
- M5: `.with_producer(&SigningKey)` assina o bloco como produtor —
  o payload `consensus` torna-se o payload de produtor assinado (sobre
  o hash de assinatura do produtor definido acima) e substitui qualquer
  `.with_consensus` anterior.

Um bloco montado passa pelo `push_block` sem recodificação (testado).

## Forks e fork choice (M3c, porta .bak)

`push_block` permanece o ponto de entrada linear (pai == tip, caso
contrário `UnknownParent`/`ParentNotTip`). `try_attach` (M3c) avalia
blocos concorrentes:

- pai desconhecido → `UnknownParent` (o bloco espera);
- linhagem válida mas atrasada → avaliação de ramo: a chain mais longa
  vence; empate perfeito → desempate determinístico (menor hash de tip na
  ordem lexicográfica — dois nós detendo os mesmos blocos sempre escolhem
  o mesmo vencedor, testado);
- **piso de finalidade**: um ramo cujo um dos ancestrais contradiz um
  checkpoint finalizado (incluído no `prev_checkpoint_hash` de um
  checkpoint aceito) é recusado (erro tipado
  `Consensus("branch contradicts a finalized checkpoint")`);
- adoção = replay desde a gênese dos blocos do ramo vencedor
  (determinismo bit a bit testado: dois nós recebendo os mesmos blocos
  em ordens diferentes terminam no mesmo estado);
- nenhum bloco recebido é canônico antes da validação completa.

## Janela RAM limitada (M7a, porta .bak)

A chain mantém apenas os **últimos 512 blocos** em RAM
(`RAM_WINDOW_BLOCKS`, porta do `HISTORY_KEEP = 512` do .bak), acima
do **piso de finalidade** (o último checkpoint finalizado):

- um bloco só sai da RAM se estiver **ambos** fora da janela
  (`height < tip − 511`) **e** estritamente abaixo do piso de
  finalidade —
  um reorg acima do piso permanece sempre possível, portanto esses
  blocos NUNCA são evacuados;
- antes do primeiro checkpoint finalizado (bootstrap), **nada é
  evacuado**: memória limitada começa com a finalidade;
- a evacuação é real: blocos, o índice de hashes (`known_hashes`, ele
  próprio limitado, indexado por altura) e txs saem da memória — o store
  (`NodeStore`, `scone-storage`) detém as cópias duráveis;
- `block(h)` abaixo da janela → `None`; `block_result(h)` → erro
  tipado `BlockPruned { height }` — os caminhos de reorg
  (`push_block`, `try_attach`, `adopt_branch`, replay de prefixo)
  o propagam: o relay deve recarregar o segmento do storage ou
  cair num sync completo (limitação documentada: nenhuma adoção
  segmentada antes da porta completa do relay);
- um bloco construído sobre um pai evacuado classifica-se como
  `BlockPruned` (não `UnknownParent`): altura do pai <
  `base_height` ⇒ o store pode servi-lo.

## Abstração de consenso

```rust
pub trait Consensus {
    fn validate_header(&self, header: &BlockHeader) -> Result<()>;
    fn validate_tx(&self, tx: &Transaction) -> Result<()>;
}
```

- `PermissiveConsensus` (padrão): aceita tudo — usado até que as
  regras reais sejam definidas;
- os hooks são chamados por `push_block`: o consenso concreto (PoW,
  dificuldade, taxas, timestamp/anti-replay, fork choice) plugará
  sem reescrever a chain;
- uma única abstração, nenhuma hierarquia de traits.

## Estado escalável (backend KV, pivô SMT, fila de expiração, pool de owners limitado)

Objetivo: 250+ bilhões de domínios potenciais — NUNCA carregar tudo em
RAM. Quatro peças:

### Backend KV (`state_backend.rs`)

```rust
pub trait StateBackend: Default + Send {
    fn get(&self, key: &Pkey) -> Option<Vec<u8>>;
    fn put(&mut self, key: Pkey, value: Vec<u8>);
    fn delete(&mut self, key: &Pkey);
    fn range(&self, from: Option<&Pkey>, limit: usize)
        -> (Vec<(Pkey, Vec<u8>)>, Option<Pkey>);
}
```

- `Pkey` = os 32 bytes brutos do `DomainId`/`TldId` — UM espaço de
  chaves ordenado (prefixos de derivação disjuntos), valores = as
  codificações `SCONE-ENTRY-DOM-V1` (88 bytes fixos) /
  `SCONE-ENTRY-TLD-V1` (variável: 33 bytes privado / 1 byte aberto,
  enum owner-option — flag + owner condicional, sem mais campo
  `until`) ;
- `MemoryBackend` (BTreeMap) = implementação de referência (dev,
  testes); NENHUMA dependência concreta em `scone-blockchain` (redb
  proibido na crate) — protocolo/consenso dependem apenas do TRAIT;
- `CachedBackend<B>` (`state_cache.rs`) = cache LRU limitado sobre um
  backend interno: write-through (o interno está sempre
  atualizado), misses nunca cacheados, `range` pagina o interno —
  mesmas respostas que o backend interno sozinho (contrato de
  determinismo, testado);
- `RedbBackend` (`scone-storage::redb_backend`) = backend persistido:
  tabela `state_kv` (chaves brutas de 32 bytes, ordem lexicográfica =
  ordem crescente exigida), contador mantido na tabela meta
  `state_kv_meta` (nunca COUNT); `CachedBackend<RedbBackend>` = estado
  com pegada RAM limitada — o "nível leve";
- `smt_persist`: consome os dirty sets do SMT
  (`take_dirty_sets`) e os escreve num backend sob chaves de prefixo
  reservado (0x01 nó, 0x02 folha) — fundação do SMT paginado;
- o trait não exige mais `Default` (um banco em disco não tem
  valor padrão sensato);
- `range` itera em ordem crescente de chave (contrato de determinismo)
  — apenas caminho de arquivo, nunca uma regra de consenso;
- o `ChainState` acessa domínios/TLDs APENAS por chave através do
  backend (leituras pontuais, escritas jornalizadas).

### Raiz SMT canônica (pivô)

- `state_root_smt()` (O(1), raiz em cache, `SCONE-STATE-SMT-V1`) é a
  **canônica** state_root: é o que `checkpoint_data` compromete e
  o que `accept_checkpoint` verifica no tip. A computação
  NUNCA atravessa o conjunto de domínios;
- o antigo fold direto `SCONE-STATE-V1` sobrevive como
  `state_root_archive()`: O(N), caminho de arquivo/verificação (refold
  completo a partir do storage para contraverificar uma raiz
  histórica) — não é um artefato de consenso. Ambos comprometem as
  mesmas folhas: estados logicamente iguais ⇒ raízes
  iguais (para ambos);
- um checkpoint aceito com a raiz SMT reverifica após um
  restore (reload): a raiz é função pura do estado restaurado
  (backend + SMT + índice).

### Fila de expiração (GC O(expirados), não O(N))

- `expirations: BTreeMap<u64, BTreeSet<DomainId>>` — instante de
  expiração → ids expirando então (ids ordenados por slot: igualdade
  de estado e ordem de drenagem independentes da ordem de chegada);
- `gc_expired_journaled(now)` drena `range(..=now)` em vez de
  varrer todo o estado: custo O(expirados naquele instante);
- limite RAM: uma entrada por domínio VIVO (expirações pendentes
  apenas — uma renovação re-chaveia o slot, o GC o drena), nunca um nó
  por domínio numa estrutura de varredura;
- mantido pelo diário de desfazer (rollback bit a bit: o slot
  restaurado é recolocado).

### Pool de owners limitado (chaves, não domínios)

- `owner_refcounts: HashMap<OwnerId, u64>` — um contador por owner
  vivo DISTINTO (domínios + TLDs), mantido pelo diário em cada
  register/transfer/revoke/assign/GC (acquire/release);
- `eligible_validators(now)` lê este índice — O(owners), NUNCA uma
  iteração sobre domínios. A vivacidade temporal já está garantida:
  o GC determinístico (timestamp do pai) removeu todo domínio
  expirado E seu refcount antes da chamada. Saída ordenada
  (determinística);
- limite RAM: O(owners distintos), independente do número de
  domínios;
- `owner_pool_root` (fold de arquivo) faz o fold desses owners
  ordenados — nenhuma varredura de domínios tampouco.

Orçamento RAM em regime permanente (backend disco/rede):
O(owners distintos + expirações pendentes + janelas de carência em
voo + diário de desfazer) — todos limitados independentemente do
número de domínios, todos restauráveis bit a bit pelo diário.

## Assentos e o comitê de finalidade (mecanismo PoS exato)

Esta seção descreve o mecanismo EXATO de elegibilidade e eleição do
comitê como implementado (`finality.rs`) — o complemento operacional
(loop de âncora, wire) vive em
`/docs/en/technical/relay.md`, a análise de segurança em
`/docs/en/technical/security-model.md`.

### Um assento = posse (nenhum registro)

Um **assento** PoS NÃO é um registro: é a posse de um
domínio ou TLD vivo. `eligible_validators(now)` computa o pool:

- chaves dos owners vivos via índice `owner_refcounts` limitado
  (um owner DISTINTO por domínio/TLD vivo — cem domínios
  sob uma chave = UM assento: o anti-Sybil econômico é por
  identidade, linear);
- um TLD conta (sua reivindicação custou um PoW — o owner de um TLD
  vivo é um stakeholder), um domínio vivo também conta
  (`valid_until > now` — garantido pelo GC determinístico,
  já drenado antes da chamada);
- **menos as chaves banidas** (M9: um equivocador slashed está fora do
  pool para sempre);
- um TLD **aberto** (`owner: None`) detém NENHUM assento: seu antigo
  owner é liberado na abertura (`owner_release` no apply do
  `SetTldOpen`), e ele morre no GC sem nunca mais votar.

Saída ordenada (ordem determinística, independente da ordem de
iteração do índice).

Consequência estrutural: **a morte do assento = a morte do
domínio/TLD**. Expiração + GC, `RevokeTld` (assento liberado),
`TransferTld`/`TransferDomain` (o assento segue o domínio ao
novo owner), abertura `SetTldOpen` (assento liberado) — toda transição
de propriedade move ou libera o assento no pool, simetricamente
ao diário `owner_refcounts` (acquire/release).

### Pool CONGELADO no último checkpoint finalizado (`FinalizedBase.pool`)

Ao aceitar um checkpoint (`accept_checkpoint`), a base de finalidade
captura o pool do estado NO BLOCO FINALIZADO:

```text
FinalizedBase { checkpoint, pool, block_ts, seed }
```

- `pool` = `eligible_validators(timestamp do bloco finalizado)`
  — a finalidade NUNCA relê o estado vivo depois disso: qualquer
  registro (reivindicação, transferência, banimento) após o checkpoint
  só conta para épocas posteriores;
- `seed` = `next_seed(signing_hash(checkpoint))` — a seed da
  PRÓXIMA época deriva do hash do **conteúdo** do checkpoint
  (com chave, conjunto de assinaturas excluído — duas visões detendo
  subconjuntos de quórum distintos do mesmo checkpoint elegem o mesmo
  comitê).

### Sorteio do comitê (bootstrap, frescor, degradação)

`committee(recovery)`:

1. **Bootstrap** (nenhum checkpoint finalizado): o pool INTEIRO assina
   — um comitê sorteado a partir de uma visão local divergiria entre
   nós atrasados (a causa raiz dos congelamentos de finalidade); o pool
   inteiro é determinístico por chain. A seleção estrita retoma na
   primeira finalidade.
2. **Regime finalizado**: sorteio do subconjunto **fresco** do pool
   congelado (RFC proof-of-service §5.2) — cada assento (chave do
   owner) é resolvido ao seu operador EFETIVO pelo estado no
   tip (delegação `SetTldOperator`/`SetDomainOperator` = fato
   on-chain recente, ou a chave do owner quando auto-operado);
   o assento participa SOB SUA CHAVE EFETIVA (é ela que
   assina), desde que tenha um **heartbeat fresco** (janela
   deslizante). Chaves banidas permanecem excluídas.
3. **Degradação segura**: menos de `MIN_FINALITY_COMMITTEE_SIZE`
   (4) operadores frescos → sorteio do pool INTEIRO (bootstrap a
   frio, queda geral de operadores — a finalidade nunca é perdida por
   construção).

`allowed_producers(block_ts)` (produção de blocos) segue o mesmo
esquema: pool inteiro no bootstrap; comitê eleito + sorteios de
recuperação (sorteio k destravado após k × `epoch_secs` de tempo de
consenso) no regime finalizado.

O quórum do checkpoint é `quorum_for(tamanho do comitê)`
(floor(2n/3)+1); um comitê abaixo de `MIN_FINALITY_COMMITTEE_SIZE`
nunca finaliza (guarda BFT).

## Adiado ao consenso

Deliberadamente indefinido nesta crate:

- **mempool**: admissão/reordenação fina (taxas, substituição)
  fica no relay (M4 trouxe teto por domínio + anti-replay TXID em
  janela — `REPLAY_WINDOW_BLOCKS = 256`);
- **ordenamento global**: a ordenação do mempool é uma ordenação local
  determinística (relay), não uma autoridade global;
- **autoridade de timestamp** além do GC: as expirações (M8b) usam o
  timestamp do bloco pai; outras regras de tempo (anti-replay fino)
  permanecem no consenso;
- armazenamento distribuído: o estado está em memória; o backend
  (`scone-storage`, redb) aplica estas regras sem duplicá-las.

Entregues desde M8b (antes adiados): PoW de registro por rede
(`RegisterTld` sempre, `RegisterDomain` num TLD aberto),
expiração/renovação 1 ano + carência de 30 dias + horizonte de 3 anos,
GC determinístico no timestamp do bloco pai, replay entre redes
impossível (gênese + campo assinado + digest PoW).

## Simulação distribuída determinística (P1.6)

`crates/scone-blockchain/src/sim/` contém um harness de simulação de
rede **determinístico** (`SimNet`): N `Blockchain`s reais (sem
mocks) dirigidos por um **relógio virtual** e uma **fila de eventos**
priorizada por `(tick, ordem de inserção)`. Sem sockets, sem async, sem
relógio de parede — cada sorteio (perda, latência, duplicação) é função
pura de `(seed, número de envio)`, de modo que um cenário com seed fixa é
reproduzível byte a byte.

### Modelo

- **Transporte**: toda emissão sofre perda probabilística
  (configurável em ‰), latência artificial limitada
  (`latency_ticks`), duplicação possível (uma cópia atrasada), e o
  mapa de partições atual (mensagens inter-grupos são perdidas).
  Duplicações nunca avançam o relógio lógico e não disparam novos
  envios: o fluxo real de eventos é idêntico com ou sem
  duplicação (contrato de determinismo, provado pelo teste
  `duplicate_and_delay`).
- **Gossip epidêmico**: um nó retransmite aos seus pares conectados
  (mesma partição) um bloco/transação/checkpoint apenas na primeira
  vista — regra de relay H2 (corta loops). Um bloco estacionado
  (órfão) NUNCA é re-broadcastado; as lacunas de recepção são
  preenchidas pelo caminho de sincronização (`GetBlocks`), como o
  relay.
- **Produção**: um passo global de produção por intervalo; cada
  grupo de partição produz no máximo UM bloco por rodada, pelo seu
  nó autorizado detendo a melhor altura. Durante o bootstrap (antes
  da primeira finalidade) o produtor é fixado no nó mais baixo: o
  conjunto de produtores autorizados deriva do estado (pool de
  domínios vivos), duas visões ligeiramente diferentes elegem comitês
  diferentes e um bloco localmente legítimo seria `InvalidProducer`
  noutro lugar — uma divergência permanente. Após a primeira
  finalidade o comitê é congelado na base de finalidade (idêntico em
  qualquer nó convergido) e a produção é livre.
- **Finalidade**: a cada múltiplo de `sign_interval` alturas, a
  âncora designada do comitê assina o conteúdo determinístico de
  seu tip via um `SignerGuard` real à prova de crash (uma chave = um
  voto por época, estado em fsync antes da difusão); os outros
  membros do comitê assinam ao receber (após verificar que o bloco
  referenciado é de fato seu bloco canônico naquela altura — guarda a
  nível de relay) e re-broadcastam o agregado enriquecido. No quórum,
  `accept_checkpoint` finaliza. Cada nó compromete-se com a primeira
  proposta vista por época (guarda bootstrap anti-dupla-finalidade:
  duas propostas concorrentes da mesma época com assinantes disjuntos
  não podem ambas acumular).
- **Crash/restart**: um nó crashado perde todo seu estado RAM (chain,
  mempool, parking, visões) mas mantém seu diretório signer-guard em
  disco, exatamente como um processo real; ele ressincroniza
  reproduzindo os blocos canônicos servidos por seus pares
  (`GetBlocks`/`GetCheckpoints`).

### Cenários cobertos (`sim::tests`)

| Teste | O que é provado |
|---|---|
| `convergence_simple` | 8 nós, 5 ‰ de perda, latência 1–5 ticks, 500 blocos: mesmo tip, mesmo `state_root_smt` em todo lugar; finalidade ativa; nunca dois checkpoints incompatíveis por época. |
| `partition_then_heal` | Partição 4a/4b por ~300 blocos: ambos os lados produzem, cura → convergência com reorg do ramo perdedor; janelas de checkpoint cruzadas nunca contradizem uma época compartilhada (antes e depois da cura). |
| `crash_restart` | Crash em h≈200, restart em h≈400: ressincronização por replay dos blocos da rede — mesma altura, tip, estado; a transação pós-crash é visível ao nó ressuscitado. |
| `duplicate_and_delay` | 10 ‰ de duplicação: altura final, raiz de estado e contagem de checkpoints finalizados idênticos à execução sem duplicação (idempotência: TxReplay/seen-sets absorvem duplicatas). |
| `concurrent_tx` | Dois owners concorrentes no mesmo domínio: exatamente uma atualização vence (o owner — `NotOwner` para o intruso, rejeição tipada limpa), nunca uma dupla aplicação de estado, a tx perdedora não entra em chain. |

### Parâmetros e limites de duração

Valores padrão: `produce_interval` 7 ticks, `sign_interval` 16
alturas, `sync_interval` 29–60 ticks (41 por padrão, rebaixado em
alguns testes), `horizon` 200k–500k ticks virtuais. Todo teste embute
uma guarda de relógio de parede < 60 s em modo debug
(`assert_under_60s`). O custo dominante é o kernel de validação
(~9 ms por `push_block_with_gc` vazio em debug, Ed25519
não otimizado): os cenários ficam sob ~8 nós × 500 blocos.

### Limites

- **Nenhuma camada de transporte real**: este harness exercita a
  lógica da chain sob entrega adversa, não o swarm (libp2p,
  request-response, backpressure, descoberta). Os testes e2e reais
  (`crates/scone-network/tests/`) permanecem a prova de transporte.
- **Sincronização em lote em RAM**: os blocos servidos vêm da janela
  RAM (cenários ficam sob `RAM_WINDOW_BLOCKS`); o caminho
  `BlockPruned` + recarga do store não é simulado (coberto pelos
  testes unitários da chain).
- **Modelo de âncora honesta**: âncoras só assinam conteúdo
  verificável contra sua própria chain canônica; o equívoco
  bizantino (dupla assinatura deliberada) é impedido pelo
  `SignerGuard` mas não exercitado como ataque ativo.
- **Uma partição por vez**: o mapa de grupos é global e alterado
  sequencialmente pelo driver; partições contínuas
  sobrepostas/evolutivas não são modeladas.
