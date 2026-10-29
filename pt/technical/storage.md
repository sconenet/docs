# Storage — persistência local do nó (redb)

> Documento normativo. Qualquer divergência entre este documento e
> `crates/scone-storage/src/` é um bug (identificar a fonte da
> verdade, corrigi-la). Implementação de referência: crate
> `scone-storage` (formato v1), backend `redb` 4.2.

## Objetivo

A persistência local embutida do nó: blocos canônicos, estados de
domínios, cache DHT, metadados. O store é uma **persistência**, nunca
uma autoridade: o estado RAM ([`ChainState`], a fonte do
consenso) é reconstruído na partida a partir do store, e toda
escrita é um **delta** (domínios modificados por um bloco), nunca uma
cópia completa do estado.

Restrição transversal (voltada ao usuário, todo milestone): **pegada
de memória mínima**. Grandes volumes (blocos, estados de domínios,
cache DHT) vivem no redb e são acessados via cursores/faixas — nunca
carregados integralmente em RAM, nunca inutilmente duplicados.

## Fronteira: o trait `NodeStore`

`crates/scone-storage/src/lib.rs` define a única face de
armazenamento visível ao resto do código (blockchain, futuro relay
M4):

- `append_block(height, hash, bytes)` /
  `append_block_with_state(..., deltas)` — append atômico (ver
  abaixo);
- `block_at_height(h)` / `block_by_hash(hash)` — leituras pontuais;
- `tip() -> (height, hash)` — O(1), lido do `meta`;
- `put_domain_state` / `domain_state` / `domain_count` —
  contador **mantido** (`meta["domain_count"]`), nunca um
  `COUNT(*)`;
- `iterate_domains(after, max) -> DomainPage` — paginação por
  cursor (lotes de 100 por padrão), ordem crescente de bytes do
  `DomainId`;
- `put_tld_state` / `tld_state` / `tld_count` / `iterate_tlds` —
  mesmos contratos para o registro de TLDs (`tld_count` mantido,
  cursor limitado) (M7d);
- `put_dht_cache` / `dht_cache` — bytes opacos de
  `SignedDnsRecord`;
- `resolve_name(canonical)` / `resolve_tld(tld)` — índice
  persistente Nome→Id (P0.2), consulta pontual O(1), nenhuma iteração
  sobre domínios (padrão `None` = backend sem índice, o chamador
  retorna a `iterate_domains` ou ao estado da chain);
- `tld_floors_all` / `snapshot_tld_floors_all` — leitura dos
  pisos de tempo de vida dos TLDs abertos (owner-option: tabelas
  dedicadas, `tld_floors_v1` viva e `snapshot_v1_floors`
  congelada);
- `rewind_to_height(h)` — reorg: remove os blocos acima de `h` e
  traz `tip`/`tip_height` de volta a `h` em UMA transação
  (preliminar de `persist_state_snapshot`, ver abaixo);
- `meta_get` / `meta_set`;
- `snapshot_meta` / `snapshot_domains` / `snapshot_tlds` — leitura
  do snapshot de estado de boot (M6b; padrões vazios = backend sem
  snapshot, `load_chain` então retorna ao carregamento completo).

Regras:

- **Nenhum tipo redb na API**: apenas bytes (`&[u8]`, `[u8; 32]`) e
  tipos do core. O backend é substituível.
- **Sem async, sem mutabilidade interior**: escritas tomam `&mut
  self` (o redb serializa os escritores), leituras tomam `&self`
  (uma transação de leitura curta por chamada).
- A API nunca revalida blocos: o hash passado a
  `append_block*` é o hash **recomputado** fornecido pelo chamador
  (`Blockchain::push_block`).

## Esquema de tabelas redb

| Tabela | Chave (ordenação) | Valor |
|---|---|---|
| `blocks_by_height` | altura `u64` (b-tree ordenado) | `hash(32) ‖ height(8 BE) ‖ canonical block` |
| `blocks_by_hash` | hash do bloco (32 B) | codificação canônica do bloco |
| `domains` | `DomainId` (32 B, ordem de bytes) | `DomainStateBytes` (abaixo) |
| `tlds` | `TldId` (32 B, ordem de bytes) | `TldStateBytes` (abaixo, M7d) |
| `dht_cache` | `DomainId` (32 B) | bytes `SignedDnsRecord` (opacos) |
| `names_v3` (P0.2) | chave de índice (32 B, ordem de bytes) | id alvo (32 B) |
| `name_reverse_v3` (P0.2) | id alvo (32 B, ordem de bytes) | `tag(1) ‖ canonical name` |
| `meta` | `&[u8]` | `&[u8]` |
| `snapshot_v1_meta` (M6b) | `u64` = 0 (slot único) | `height(8 BE) ‖ tip_hash(32)` = exatamente 40 B |
| `snapshot_v1_domains` (M6b) | `DomainId` (32 B, ordem de bytes) | `DomainStateBytes` (mesmo formato que `domains`) |
| `snapshot_v1_tlds` (M6b) | `TldId` (32 B, ordem de bytes) | `TldStateBytes` (mesmo formato que `tlds`) |
| `tld_floors_v1` (owner-option) | `TldId` (32 B, ordem de bytes) | `u64` (piso de expiração, BE implícito do redb) |
| `snapshot_v1_floors` (M6b) | `TldId` (32 B, ordem de bytes) | `u64` — cópia congelada dos pisos (estado NO snapshot, não no tip) |

Chave `meta` adicional (P0.3/P0.4):

| Chave | Valor |
|---|---|
| `verified_snapshot_v1` (P0.3) | `height(8 BE) ‖ tip_hash(32) ‖ manifest_hash(32)` = 72 B — marcador de um snapshot verificado importado (reservada, como `tip`) |

Chaves `meta`:

| Chave | Valor |
|---|---|
| `format_version` | `u64 BE` = `STORAGE_FORMAT_VERSION` (1) |
| `tip` | hash (32 B) do bloco tip |
| `tip_height` | `u64 BE` (torna `tip()` O(1)) |
| `domain_count` | `u64 BE` (contador mantido) |
| `tld_count` | `u64 BE` (contador mantido, M7d) |

O header `hash ‖ height` de `blocks_by_height` torna cada índice
autossuficiente (verificável sem ler o bloco); ambos os índices são
atualizados na mesma transação.

### Convenção de gênese

O bloco gênese é determinístico (constantes de protocolo) e
**nunca é armazenado** como bytes. Um store novo tem `tip == (0,
genesis_hash)`; o primeiro bloco anexado tem altura 1. O hash de
gênese é sempre rederivado
(`scone_blockchain::genesis_hash()`), nunca lido.

## Formato de codificação `DomainStateBytes` (M8b)

65 ou 97 bytes conforme a tag de versão inicial, inteiros de largura
fixa big-endian (canônico, comparável byte a byte):

```text
DomainStateBytes = tag ‖ owner(32) ‖ sequence(8 BE) ‖ registered_at(8 BE)
                   ‖ valid_until(8 BE) ‖ mutations(8 BE) [‖ record_hash(32)]

tag = 0x01 : record_hash presente → exatamente 97 bytes
tag = 0x02 : record_hash ausente  → exatamente 65 bytes
```

- `registered_at` / `valid_until` (M8b): a expiração do
  registro (1 ano na reivindicação, renovável, ver
  `/docs/en/technical/blockchain.md`).
- `mutations` (V3): o contador de mutações monótono do namespace
  (geração CAS anti-replay — ver `transactions.md`, O contador de
  mutações). As codificações M8b (57/89 bytes, sem contador) são
  **ilegítimas**: decodificação estrita → `Corrupted` (reset
  testnet; um store pré-contador regenera via replay).

As codificações M7 (41/73 bytes, sem expiração) são
**ilegítimas** desde M8b: decodificação estrita → `Corrupted`. (Um
store M7 regenera via replay: `load_chain_replay`.)

- `owner`: `OwnerId` (32 B);
- `sequence`: a sequência do último `UpdateDomain` aplicado (0 logo
  após um `RegisterDomain`);
- `record_hash`: o compromisso on-chain atual do conjunto de
  registros DNS (`None` enquanto nenhum `UpdateDomain` foi
  aplicado).

Decodificação **estrita** (`DomainStateBytes::decode`): tag
desconhecida, comprimento errado ou bytes em excesso →
`StorageError::Corrupted`, nunca um pânico.

## Formato de codificação `TldStateBytes` (owner-option)

Comprimento variável conforme a variante do enum — 50 bytes para um
TLD **privado**, 10 para um TLD **aberto** (nem sequência nem
hash de registro):

```text
TldStateBytes = tag(0x04) ‖ flag(0x00|0x01) ‖ owner(32)? ‖ until(8)
                [‖ mutations(8)]
```

- `flag` (owner-option, enum `TldState`): `0x00` =
  `TldState::Open` (de ninguém — autoatendimento + PoW, sem bytes
  de owner), `0x01` = `TldState::Private` (32 bytes de owner + 8 bytes de
  contador se seguem). Estritamente canônico — qualquer outro byte →
  `Corrupted`. O inváriante "aberto ⟺ sem owner" é ESTRUTURAL: os
  dois estados não podem ser confundidos.
- **`until` (8 bytes, BE64)**: o relógio próprio do TLD — o fim do
  mandato privado (reivindicação/renovação + 3 anos) ou a agenda de
  morte aberta (piso de 1 ano no nascimento, empurrado por cada
  domínio vivo). É um
  atributo da entrada novamente (mudança de ciclo de vida dos TLDs).
  A tabela derivada `tld_floors_v1` (+ sua cópia congelada
  `snapshot_v1_floors`) ainda rastreia os pisos, reconstruída no
  replay e restaurada no boot — um nó reiniciado mata os mesmos
  TLDs que um nó contínuo.

Os layouts pré-contador tag `0x04` (42/2 bytes, sem contador), as
codificações M7d (33 bytes, tag `0x01`), M8b (34 bytes, tag
`0x02`, booleano `open`) e a tag legada `0x03` (`until` embutido)
são **ilegítimas**: decodificação estrita → `Corrupted`
(pré-mainnet, a regeneração via replay é o caminho de migração).

A tag existe apenas para a evolução do formato: um layout futuro
eleva a tag e os leitores antigos falham com `Corrupted` em vez de
adivinhar. Decodificação **estrita** (`TldStateBytes::decode`): tag
desconhecida, comprimento errado, bytes em excesso ou flag não
canônica → `StorageError::Corrupted`, nunca um pânico.

## Garantias de atomicidade

`append_block_with_state` escreve em **uma única**
`WriteTransaction` redb:

1. o bloco em `blocks_by_height` (checagem prévia: a altura não
   deve existir com conteúdo diferente, caso contrário
   `StorageError::Conflict`);
2. o bloco em `blocks_by_hash`;
3. cada delta de estado de domínio (`domains`);
4. o contador `domain_count` (incrementado apenas pelo número de
   domínios **novos**; um `UpdateDomain` de um domínio existente
   não o incrementa);
5. cada delta do registro de TLDs (`tlds`, M8b: reivindicações,
   transferências, abrir/fechar) — **na mesma transação**: um bloco
   escrito sem suas mudanças de TLD é estruturalmente impossível;
6. o contador `tld_count` (apenas TLDs novos);
7. cada **remoção** (M8b): domínios expirados pelo GC
   determinístico (`removed_domains`) e TLDs revogados
   (`removed_tlds`) saem do store na mesma transação, contadores
   decrementados — um restart jamais pode ressuscitar um registro
   expirado;
8. os deltas do **índice de nomes** da P0.2 (upserts e remoções,
   ver abaixo) — mesmas garantias: um crash jamais deixa uma entrada
   de índice apontando para um domínio ausente;
9. `tip` e `tip_height`.

O redb commita estilo WAL: um crash no meio da escrita deixa o estado
coerente **anterior**. **Um bloco escrito sem seu estado é portanto
estruturalmente impossível** — este é o inváriante-chave do
milestone.

Garantias adicionais:

- **Monotonicidade**: append numa altura ≠ `tip_height + 1` →
  `StorageError::NonMonotonicHeight`, nada é escrito (verificado
  *antes* de abrir a transação de escrita);
- **Idempotência**: re-anexar o bloco tip atual com o mesmo par
  `(height, hash)` é no-op (o caminho rápido compara altura e hash;
  em altura igual, bytes de bloco idênticos são igualmente um
  no-op silencioso, bytes diferentes um `Conflict`);
- **Banco corrompido**: arquivo truncado/poluído → erro tipado
  (`StorageError::Database`/`Corrupted`) na abertura ou na leitura,
  nunca um pânico (testado);
- **Versão de formato**: um `format_version` ≠ 1 →
  `StorageError::UnsupportedFormat` (um caminho de migração
  explícito é exigido, nunca adivinhar).

## Integração com a blockchain (`integration.rs`)

Separação de papéis: [`ChainState`] em RAM = autoridade de
consenso; store = persistência. A estratégia escolhida (a mais
simples, como documentado): **persistir o estado a cada bloco** —
`store_block` escreve o bloco + os estados dos domínios **tocados
por esse bloco** (delta). O replay completo permanece possível e
serve de caminho de controle/reparo.

- `store_block(store, chain, block, hash)`: codifica o bloco
  canonicamente, lê o estado final dos domínios tocados E dos TLDs
  reivindicados do estado RAM (já atualizado pelo
  `push_block`), append atômico.
- `load_chain(store)`: **sem replay** — lê tip + bloco tip
  (leituras de bloco O(1)), restaura o estado dos domínios em
  páginas de 100 (`DOMAIN_PAGE`) **e o registro de TLDs em páginas
  de 100** (`TLD_PAGE`, M7d — fechando a janela de restart: sem
  isso, um nó reiniciado esquecia todo TLD reivindicado e rejeitava
  todo `RegisterDomain` sob esse TLD com `UnknownTld`, embora seu
  disco contivesse o estado completo). A chain RAM mantém apenas
  gênese + tip; blocos históricos ficam no store e são
  servidos a partir dele. Desde M8, `load_chain` reconstrói depois
  o **índice anti-replay TXID** (ver abaixo).
- `load_chain_replay(store)`: reproduz todos os blocos armazenados
  um a um (nunca todos em RAM) com validação completa; o tip
  resultante deve igualar o tip armazenado, caso contrário
  `Corrupted`.
- `rebuild_replay_index_from_store(store, chain, tip_height)`
  (M8): re-escaneia os últimos
  `min(REPLAY_WINDOW_BLOCKS, tip_height)` blocos persistidos (um
  bloco em RAM por vez) e reinsere cada TxId no índice da chain
  **na sua altura de inclusão original**, via a API pública
  `Blockchain::rebuild_replay_index` (sem acesso a campos
  privados). Bloco ausente sob o tip ou indecodificável dentro da
  janela → `Corrupted` (esses bytes já serviram para construir a
  chain: uma falha aqui sinaliza um store danificado).
- `touched_domains(block)`: ids dos domínios modificados pelas
  transações do bloco, deduplicados, ordem do bloco.
- `touched_tlds(block)`: ids dos TLDs mutados pelo bloco
  (`RegisterTld`, `TransferTld`, `RevokeTld`, `SetTldOpen`),
  deduplicados, ordem do bloco (M8b).
- `store_block_with_removals(...)`: variante do `store_block`
  recebendo também os domínios removidos pelo GC do bloco (M8b)
  para a escrita atômica descrita acima. O relay alimenta essa lista
  a partir de `Blockchain::push_block_with_gc` (o resultado da
  aplicação expõe `gc_removed_domains`). **Evacuação do GC no
  cache DHT**: as entradas `dht_cache` dos domínios removidos são
  apagadas na MESMA transação que o bloco (os deltas de append
  carregam a tabela `dht_cache`) — um registro DHT nunca sobrevive
  ao domínio que serviu, entradas expiradas nunca se acumulam para
  sempre (teste: `gc_evicts_the_dht_cache_entry`).

O relay (M4) usará: `load_chain` na partida, depois para cada
bloco aceito `push_block` (RAM) depois `store_block` (disco).

## Persistência de reorg (`rewind_to_height` + `persist_state_snapshot`)

Uma reorg (mudança para um ramo irmão mais forte) não pode passar
pelo append monotônico — escrever o sufixo vencedor falharia com
`NonMonotonicHeight`. O caminho dedicado (correção
sticky-minority, relay + store):

1. `rewind_to_height(fork_height)` — UMA transação redb remove
   os blocos acima do ponto de fork e traz `tip`/`tip_height` de
   volta (o índice de nomes e os estados vivos são consertados na
   próxima etapa);
2. `persist_state_snapshot(store, chain)` — reescreve as tabelas
   vivas (`domains`, `tlds`, índice de nomes, pisos de TLD) a partir
   do export do estado RAM pós-reorg: as tabelas vivas NUNCA
   descrevem o ramo PERDEDOR no boot;
3. re-append monotônico do sufixo adotado (deltas ordinários).

Uma falha de persistência neste caminho é logada e degradada (o
store cura no boot via replay) — nunca fatal.

## Snapshot de estado de boot (M6b)

`load_chain` como descrito acima ainda é O(tamanho do estado):
nenhum bloco é reproduzido, mas toda a tabela `domains` é relida. É
o caminho estado-persistido-por-bloco que torna o replay
desnecessário. O snapshot M6b aborda um custo diferente: o caso em
que o estado RAM deve ser reconstruído **por replay** (reparo,
controle, ou um backend sem estado persistido por bloco). Ele
oferece um boot em **O(tip − H)** — reproduzindo apenas o sufixo
acima do snapshot.

### Escrita

- **Intervalo**: `SNAPSHOT_INTERVAL = 64` (constante pública de
  `scone-storage`). O gatilho é o append do bloco
  `k·SNAPSHOT_INTERVAL + 1`: essa transação congela o estado das
  tabelas vivas **tal como está antes de aplicar esse bloco**, ou
  seja, o estado canônico após o bloco `k·SNAPSHOT_INTERVAL`.
- **Nunca no tip**: o snapshot portanto fica no máximo a `tip − 1`,
  sempre estritamente abaixo do tip no momento em que é
  escrito. Um crash logo depois jamais pode deixar um snapshot
  "acima" da chain persistida.
- **Atômico**: o congelamento (limpar + copiar as tabelas
  `domains`/`tlds` para `snapshot_v1_*`, escrever
  `snapshot_v1_meta`) acontece na MESMA transação redb que o
  append do bloco gatilho — um crash deixa ou o snapshot antigo
  intacto, ou o novo completo com seu bloco (WAL do redb).
- **Slot único**: todo snapshot SUBSTITUI o anterior
  (tabelas esvaziadas depois recopiadas) — sem acumulação, sem poda
  para gerenciar, a pegada em disco é limitada por ~2× o estado.
- O formato dos estados congelados é estritamente o de
  `DomainStateBytes`/`TldStateBytes` (reuso, nenhuma nova
  codificação); apenas a linha `snapshot_v1_meta` é um formato novo
  (`height(8 BE) ‖ tip_hash(32)`, decodificação estrita: qualquer
  outro comprimento → `Corrupted`).

### Leitura no boot (`load_chain`)

1. O tip é validado como sempre (hash recomputado do header do
   bloco tip vs `meta["tip"]`).
2. Se `snapshot_meta()` existe, sua altura `H` satisfaz
   `0 < H < tip_height`, e o hash RECOMPUTADO do bloco
   armazenado em `H` iguala o `tip_hash` do snapshot (**âncora**
   — um snapshot estranho à chain persistida, alterado, ou de outro
   arquivo é detectado aqui), o estado congelado é restaurado em
   páginas de 100 e depois apenas os blocos `H+1..=tip` são
   reproduzidos (validação completa, idêntica aos blocos vivos).
3. O tip atingido pelo replay deve igualar o tip armazenado.
4. **Qualquer falha neste caminho (snapshot ausente, altura
   inutilizável, âncora inválida, bytes indecodificáveis, replay
   falho) degrada silenciosamente para o carregamento completo
   pré-M6b** (leitura das tabelas vivas, sem replay) — um snapshot
   ruim jamais falha o boot. A compatibilidade retroativa é o caso
   particular "sem snapshot": um store escrito antes de M6b (sem as
   tabelas `snapshot_v1_*`) carrega identicamente, `snapshot_meta()`
   sendo `None`.

O sufixo reproduzido é limitado por `SNAPSHOT_INTERVAL` (no máximo
64 blocos no momento do gatilho + os blocos minerados desde então):
o boot é O(tip − H) com H garantidamente recente.

## Índice anti-replay TXID reconstruído no boot (M8)

O índice anti-replay (`included`, TxId → altura de inclusão, janela
`REPLAY_WINDOW_BLOCKS = 256` — ver
`/docs/en/technical/blockchain.md`) é uma **função pura da chain
canônica**: ele nunca é persistido. Como `load_chain` não reproduz
o histórico, uma chain restaurada partia antes de M8 com um índice
**vazio** — um nó reiniciado aceitava a reinclusão de qualquer
transação ainda dentro de sua janela de replay (limitação
documentada; as regras de estado permaneciam a única salvaguarda).

M8 fecha esta janela: após a restauração (caminho completo OU
snapshot + replay do sufixo), `load_chain` chama
`rebuild_replay_index_from_store`, que relê os últimos
`min(REPLAY_WINDOW_BLOCKS, tip_height)` blocos persistidos — um
bloco em RAM por vez, memória limitada como em todo lugar — e
reinseri cada TxId na sua **altura original**. Um store com menos
de 256 blocos portanto reconstrói seu índice inteiro.

Propriedades:

- **Decisões idênticas ao nó vivo**: mesmas entradas, mesma regra
  de poda (`h > tip − 256`) — um nó que reinicia e um nó que nunca
  parou rejeitam os mesmos replays (testado: contagem exata do
  índice e igualdade com o índice vivo podado).
- **Semântica de poda inalterada**: uma transação mais velha que a
  janela permanece admitida pelo índice (e depois falha nas regras
  de estado se aplicável) — exatamente como antes.
- **O snapshot NÃO congela o índice**: o replay do sufixo do
  snapshot reconstrói as entradas acima de H, o rescan M8
  acrescenta a parte da janela ABAIXO de H (mesmo contrato que
  owner_keys/grace: reconstruído, nunca persistido).
- **Custo**: ≤ 256 leituras pontuais de bloco no boot, O(janela) —
  independente da idade da chain.
- Um bloco ausente ou indecodificável **dentro da janela** falha o
  boot com `Corrupted` (esses bytes já serviram para construir a
  chain); fora da janela, os blocos históricos ficam intactos.

`load_chain_replay` (zero confiança) reconstrói o índice do mesmo
modo através do seu replay completo — os dois caminhos convergem.

## Índice persistente Nome → Id (P0.2)

`DomainId = BLAKE3-256(DOMAIN_ID_VERSION ‖ canonical name)` não é
invertível: resolver um nome (relay/DNS) exige um índice
persistente. A P0.2 o acrescenta ao store, com a mesma disciplina
transacional do resto.

### Esquema

Duas tabelas companheiras, escritas e apagadas juntas:

```text
names_v3        key  = BLAKE3-256("SCONE-NAME-IDX-V1" ‖ canonical name)  (32 B)
                val  = target id (32 B : DomainId or TldId)

name_reverse_v3 key  = target id (32 B)
                val  = tag(0x01 domain | 0x02 TLD) ‖ canonical name
```

- A chave de `names_v3` é um **hash do nome, não o nome em texto
  claro** (tabelas de índice compactas, nenhum gossip de nomes via
  dumps). A chave é uma **função pura do nome**: o índice é
  reconstruível a qualquer momento a partir da chain
  (`rebuild_name_index`), nenhum metadado adicional é persistido
  para ele.
- `name_reverse_v3` armazena o nome (exatamente uma vez) porque os
  caminhos de remoção (GC de domínios expirados, `RevokeTld`) só
  conhecem **ids**: é preciso de algum modo encontrar a chave
  hasheada a apagar. A tag distingue os dois namespaces.
- Separação estrita: `resolve_name` NUNCA responde uma entrada de
  TLD e `resolve_tld` nunca uma entrada de domínio (a tag da tabela
  reversa decide; um `DomainName` tem ≥ 2 labels, um TLD tem 1 — as
  duas derivações de chave partilham o espaço `names_v3` mas não os
  resultados).
- O `DomainId` permanece 32 bytes, nunca reduzido (regra dura do
  projeto: identidade de consenso, resistência a colisões).

### Consistência transacional

Upserts (`RegisterDomain`, `AssignDomain`, `RegisterTld` — nomes
carregados em texto claro pelas transações) e remoções (GC,
revogações — ids puros) são carregados pelo `StateDelta` do bloco
(`name_upserts` / `name_removals`) e aplicados na **mesma transação
redb** que as escritas do bloco e do estado. Um crash portanto não
pode deixar nem uma entrada de índice apontando para um domínio
ausente (nenhum fantasma), nem um domínio registrado sem sua
entrada. Um append falho não escreve nada (testado). As variantes
nomeadas `put_domain_state_with_name` / `put_tld_state_with_name` /
`remove_domain_state` oferecem a mesma atomicidade às ferramentas
de reparo.

`store_block` / `store_block_with_removals` derivam os deltas de
nomes das MESMAS transações do bloco — nenhum parâmetro extra na
interface do relay.

### Reconstrução (`RedbStore::rebuild_name_index`)

O índice é função pura da chain canônica, como o índice
anti-replay M8. `rebuild_name_index` o reconstrói a partir dos
**blocos persistidos** (os `DomainStateBytes` NÃO contêm o
nome — apenas owner/sequência/expiração — portanto a fonte da
verdade é o fluxo de transações, onde `RegisterDomain` e
`AssignDomain` carregam o nome canônico em texto claro):

1. passagem 1: escanear `blocks_by_height` gênese→tip, um bloco em
   RAM por vez; coletar as amarrações `nome → id` vistas em
   `RegisterDomain`/`AssignDomain`/`RegisterTld`;
2. passagem 2: filtrar apenas alvos **vivos** (o estado persistido
   é autoritativo) — um domínio registrado depois GC'd/expirado não
   é reindexado;
3. passagem 3: apagar + reescrita completa das duas tabelas em UMA
   transação (crash → índice antigo ou novo completo).

Um bloco ausente ou indecodificável → `Corrupted` (mesmos bytes que
construíram a chain). O custo é O(nomes distintos vistos) em RAM
(≤ 253 B por nome) e O(blocos) em leituras — uma ferramenta de
reparo, não um caminho de boot.

### O que o índice NÃO é

- Não é uma autoridade: o estado da chain RAM permanece a única
  autoridade; o índice é um cache reconstruível (uma divergência é
  um bug de transacionalidade, detectável por `rebuild_name_index`
  comparado a ele).
- Não é congelado no snapshot M6b: as tabelas `names_v3`/
  `name_reverse_v3` são vivas e seguem os deltas de cada bloco; o
  snapshot de estado não as embute (reconstrução por deltas na
  retomada, ou `rebuild_name_index` como reparo).

### O que o snapshot NÃO compromete

O estado RAM `ChainState` carrega estruturas internas não
comprometidas pelo consenso (chaves de owners, janelas de
carência). Elas NÃO são congeladas: o replay do sufixo as
reconstrói aplicando as transações (determinístico, mesmo conteúdo
que o replay completo para a janela pertinente). O índice
anti-replay TXID segue o mesmo contrato (M8, ver seção anterior):
reconstruído no boot, nunca congelado. A igualdade bit a bit com
`load_chain_replay` é verificada por teste sobre o estado canônico
(state_root + SMT).

## Snapshot de estado VERIFICADO — chunks + manifesto + verificação criptográfica (P0.3)

O snapshot M6b acima acelera o boot de um nó que confia no seu
disco. A P0.3 aborda uma necessidade diferente: transportar um
estado entre nós (bootstrap de um novo nó, reparo) sem confiar no
transporte. O formato e a verificação vivem em
`crates/scone-storage/src/verified_snapshot.rs` + a reconstrução
da raiz em `crates/scone-blockchain/src/snapshot_verify.rs`.

### Modelo de confiança

O snapshot carrega o estado de um bloco **finalizado** — um
checkpoint assinado pelo quórum do comitê
(`CheckpointData.state_root`, a raiz O(1) `state_root_smt`). A
autenticidade do checkpoint (quórum de assinaturas Ed25519) é
validada pela camada da chain ANTES da importação. A importação em
si verifica a aderência snapshot ↔ checkpoint por RECOMPUTAÇÃO:

1. **hash de cada página**: `blake3("SCONE-SNAP-PAGE-HASH" ‖
   codificação completa da página)` deve igualar o hash anunciado
   pelo manifesto (um byte falsificado → rejeição);
2. **soma das entradas == contadores do manifesto** (uma página
   ausente ou um contador errado → rejeição);
3. **raiz SMT recomputada a partir das entradas importadas ==
   `checkpoint.data.state_root`** — A verificação criptográfica:
   todo o estado (domínios + TLDs) é re-dobrado num SMT fresco
   (`snapshot_verify::recompute_state_root`, folhas
   `SCONE-LEAF-DOM-V1`/`SCONE-LEAF-TLD-V1` exatamente como o
   caminho vivo) e a raiz deve igualar a que o comitê assinou. Um
   snapshot falsificado, truncado ou incompleto é rejeitado ANTES DE
   QUALQUER ESCRITA;
4. **consistências estruturais**: `height`/`tip_hash`/`state_root`
   do manifesto == os do checkpoint; ids estritamente
   crescentes (sem duplicatas); índices de página sequenciais;
   limites DoS (≤ 10.000 entradas/página, ≤ 2^20 páginas, orçamento
   total).

Falha de verificação → `StorageError::SnapshotRejected`, o store
permanece estritamente inalterado (tudo é verificado antes da
transação de escrita).

### Formato serializado (estrito, big-endian, decodificação impiedosa)

```text
manifest = "SCONE-SNAP-MAN-V1"
        ‖ height(8) ‖ tip_hash(32) ‖ state_root(32)
        ‖ page_count(4) ‖ domain_count(8) ‖ tld_count(8)
        ‖ page_hashes(page_count × 32)

page    = "SCONE-SNAP-PAGE-V1"
        ‖ index(4) ‖ height(8) ‖ state_root(32)      ← every page is
        ‖ entry_count(4) ‖ entries                   self-identifying

entry   = kind(0x01 domain | 0x02 TLD | 0x03 TLD floor) ‖ id(32)
          ‖ [len(2) ‖ state | floor(8 BE)]
          (state = DomainStateBytes 57/89 B or TldStateBytes 2/34 B —
           the existing canonical storage formats, reused; kind
           0x03 (lifetime floor of an open TLD, owner-option) has
           a dedicated format WITHOUT a length prefix:
           id(32) ‖ floor(8 BE), fixed size)
```

Ordem das páginas: todas as entradas de domínio (ids crescentes),
depois todas as entradas de TLD (ids crescentes), depois os pisos
de tempo de vida dos TLDs CONGELADOS na altura do snapshot
(`snapshot_tld_floors_all` — não a tabela viva) — a ordem da
paginação por cursor do store. Cada página carrega `(index,
height, state_root do checkpoint, entries)`: uma página isolada
sabe a que snapshot pertence. Páginas de 256 entradas na
exportação (`VERIFIED_SNAPSHOT_PAGE_ENTRIES`); limites DoS na
importação. Os pisos de TLD (`tld_floors_v1`) fazem parte do
formato portátil: o tipo 0x03 os carrega, e a importação os exige
estritamente crescentes e mirando um TLD PRESENTE no snapshot —
caso contrário o importador jamais mataria o que um exportador
mataria (fork silencioso, rejeitado).

### Exportação (`export_verified_snapshot(store, checkpoint)`)

Serializa o estado do ÚLTIMO CHECKPOINT FINALIZADO — não do
tip. Emite domínios, TLDs, depois os pisos de tempo de vida dos
TLDs congelados na altura do snapshot (simetria
export/import: a exportação historicamente omitia os pisos —
corrigido, teste de regressão
`export_carries_tld_lifetime_floors`). Pré-requisito: o store
detém um snapshot M6b persistido (`snapshot_v1_*`) exatamente na
altura do checkpoint (a janela
`CHECKPOINT_KEEP`/`SNAPSHOT_INTERVAL` garante que tal estado está
regularmente disponível); a âncora do bloco é RECOMPUTADA a partir
dos bytes armazenados e deve igualar
`checkpoint.data.block_hash`. Caso contrário →
`StorageError::SnapshotUnavailable` (a exportação é uma ação
explícita: esperar o próximo intervalo/checkpoint).

### Importação (`import_verified_snapshot(store, checkpoint, manifest, pages)`)

Verifica tudo (ver acima), depois escreve em UMA transação redb:
tabelas vivas `domains`/`tlds` + contadores + `tip = (H,
hash do checkpoint)`, tabelas congeladas `snapshot_v1_*` ancoradas
no mesmo bloco, e o marcador reservado
`meta["verified_snapshot_v1"]` (`height ‖ tip_hash ‖
manifest_hash`). Alvo: apenas store vazio (caso contrário
`SnapshotUnavailable`).

### Custo de verificação (documentado)

Reconstrução do SMT via inserções ordenadas: N entradas × O(40)
hashes ≈ O(N·40). ~2.1 M hashes blake3 para 100.000 domínios
(dezenas de ms). PONTUAL — uma importação de bootstrap/reparo,
nunca um caminho por bloco: aceitável. A inserção ordenada (ids
crescentes) cria cada ponto de ramificação uma única vez; em todo
caso a raiz depende apenas do conteúdo (garantia SMT, fuzz
diferencial em `smt.rs`).

## Bootstrap de estado — sem replay da gênese (P0.4)

Um novo nó entra na rede sem reproduzir o histórico desde a
gênese:

```text
finalized checkpoint (relay, validated by the chain layer)
        │
        ▼
verified snapshot (manifest + pages, crypto verification above)
        │  import → state at H + tip (H, hash) + marker
        ▼
anchor block H (requested from the network BY HASH — append_bootstrap_anchor)
        │
        ▼
recent blocks H+1..H+k (relay, increasing height, ordinary atomic
        │               append — the storage API requires only this)
        ▼
synced — boot replays ONLY the suffix above H
```

- **API de storage exposta ao relay**: `append_bootstrap_anchor(H,
  hash, bytes)` (lança o bloco âncora: marcador exigido,
  consistência altura/hash com o marcador, idempotente) depois
  append atômico ordinário por altura crescente. A recuperação pela
  rede (por hash a partir do manifesto) é uma preocupação do
  relay, não do store.
- **`load_chain`** reconhece o marcador: o histórico abaixo de H é
  legitimamente ausente (nó não-arquivo). Concretamente: o rescan
  da janela anti-replay M8 tem piso em H (blocos abaixo de H não
  existem e não estão faltando); um tip sem bloco armazenado DEPOIS
  de uma importação (âncora ainda não lançada) → diagnóstico
  tipado `SnapshotUnavailable` ("a âncora deve vir da rede"), não
  corrupção.
- **Nó arquivo vs nó de estado completo**: um nó arquivo (flag de
  operador) guarda tudo — serve blocos históricos à rede e dá boot
  pelos caminhos existentes. Um nó de estado completo (padrão P0.4
  para um novo nó CONFIGURADO assim) dá boot em H: valida blocos
  futuros (replay do sufixo) mas não pode servir o histórico abaixo
  de H nem reproduzir a gênese. **O comportamento padrão NÃO
  muda**: este caminho só está ativo se o operador disparar
  explicitamente a importação (`import_verified_snapshot` é uma
  ação dedicada, nunca automática); todo nó existente dá boot
  identicamente.
- **Igualdade com o nó completo**: após juntar-se ao sufixo
  H+1..H+k, o estado, o tip e o `state_root_smt` são bit a bit
  iguais a um nó completo no mesmo tip (testado); o índice
  anti-replay cobre exatamente as txs em/acima de H — um nó
  não-arquivo não pode conhecer a janela abaixo de H, as regras de
  estado permanecem a salvaguarda (uma tx duplicada abaixo de H
  falha no estado, nunca aceita duas vezes).

## Política de memória

- **Cursores em todo lugar**: `iterate_domains` limita a página a
  `max` entradas (100 por padrão) via `range` preguiçoso do redb; a
  tabela inteira de domínios nunca é materializada. Grandes listas
  (TLDs, domínios) passam obrigatoriamente por este cursor
  (`iterate_tlds`, mesmo contrato, M7d).
- **Apenas deltas**: `append_block_with_state` escreve apenas os
  domínios e TLDs modificados pelo bloco — nunca o estado completo;
- **Nenhuma duplicação síncrona**: o estado RAM não é
  re-persistido por bloco; o estado em disco não é recarregado por
  bloco;
- **Leituras pontuais**: toda leitura abre uma transação de leitura
  curta, retorna o valor, fecha; os `AccessGuard` do redb vivem
  apenas pela duração da cópia limitada;
- **Blocos históricos nunca em RAM**: `load_chain` carrega apenas
  o tip; `load_chain_replay` faz streaming altura por altura;
- **Contador mantido**: `domain_count` é O(1), não uma varredura.

## Limites DoS

O store limita o que um par hostil pode fazê-lo persistir ou
alocar:

- **`MAX_DHT_CACHE_ENTRY` (64 KiB)**: um registro DHT maior é
  rejeitado antes de qualquer transação de escrita por
  `put_dht_cache` → `StorageError::TooLarge` (nada é escrito);
- **`MAX_DOMAIN_PAGE` (10.000)**: `iterate_domains`
  *internamente* limita a página pedida a `min(max, 10_000)` — um
  `max` excessivo jamais materializa mais de 10.000 estados em RAM;
  `iterate_tlds` tem o mesmo limite (`MAX_TLD_PAGE`, M7d);
- **Chaves `meta` reservadas**: `meta_set` rejeita `tip`,
  `tip_height`, `format_version`, `domain_count`, `tld_count` e
  `verified_snapshot_v1` (P0.3) com
  `StorageError::ReservedKey` — sobrescrever a contabilidade
  interna do store o corromperia silenciosamente.
- **Cache DHT duplamente limitado (LRU)**:
  `DHT_CACHE_MAX_ENTRIES` (100.000 entradas) E
  `DHT_CACHE_MAX_BYTES` (2 GiB — guarda do caso hostil; cada
  entrada já é limitada a 64 KiB por `MAX_DHT_CACHE_ENTRY`), o
  limite que atingir primeiro evacua pela ordem de chave. Os
  contadores (entradas + bytes) são mantidos em tempo de escrita —
  nunca uma varredura para contar; a reconstrução exata acontece
  apenas na abertura (a única varredura completa, no boot).
  Acessores `dht_cache_entries`/`dht_cache_bytes`, limites
  ajustáveis via `set_dht_cache_bounds` (teste:
  `dht_cache_respects_the_double_bound`).

## Modelo de confiança

Dois caminhos de carregamento, dois níveis de garantia:

- **`load_chain` — confiança no disco local**: o hash do tip é
  **recomputado** a partir do header do bloco tip e comparado ao
  `meta["tip"]` armazenado (divergência →
  `StorageError::Corrupted`, nunca uma chain restaurada sobre um
  hash forjado), e os estados de domínios são estritamente
  decodificados. Contudo, os blocos históricos **não são
  reproduzidos**: os estados persistidos são aceitos ao pé da
  letra. Este é o boot rápido de um nó que confia no próprio disco
  (escrito apenas por esse mesmo nó, atomicamente). Desde M6b,
  este caminho usa primeiro o **snapshot de estado** quando
  utilizável (ver abaixo): a âncora do snapshot é recomputada e o
  sufixo acima de H é revalidado como blocos vivos — a parte
  reproduzida é portanto zero-confiança, a base congelada
  confiança-disco.
- **`load_chain_replay` — zero confiança**: todos os blocos são
  relidos e revalidados um a um (validação completa, idêntica aos
  blocos vivos); o tip resultante deve igualar o tip armazenado.
  Esta é a ferramenta de controle/reparo.

**Recomendação**: um nó estrito (ou um nó cujo arquivo de
armazenamento possa ter sido adulterado — disco partilhado, restore
de um backup duvidoso) deve partir via `load_chain_replay`, não
`load_chain`.

## Concorrência

O redb 4.x permite **um único objeto `Database` por arquivo e por
processo** (duplo `open` simultâneo falha com
`DatabaseAlreadyOpen`). O modelo suportado: um `RedbStore`
clonado (`Arc<Database>`) partilhado entre threads — escritores
serializados por `&mut self` + trava interna do redb, leitores
concorrentes (transações de leitura snapshot). Depois que todos os
handles são totalmente derrubados, o arquivo pode ser reaberto.

## Testes (`tests/redb_store.rs` + módulos `#[cfg(test)]`)

Todo teste usa seu próprio tmpfile redb (`tempfile`). Cobertura:

- roundtrip de blocos gênese→N, restart, releitura, replay;
- M7d: registro de TLDs persistido por delta atômico, restaurado no
  restart (um `RegisterDomain` sob um TLD reivindicado antes do
  restart permanece admissível — janela fechada), contadores exatos
  na reabertura, cursor `tlds` crescente, append falho não deixa
  TLD, chave reservada `tld_count` rejeitada;
- atomicidade: append falho não deixa nada (tip, domínios, índice
  intactos); deltas duplicados contados uma vez;
- cursores de domínios: paginação 100/7, ordem crescente de
  `DomainId`, início estritamente após um id existente ou ausente,
  `max = 0`;
- contador `domain_count` exato (insert/update/reopen);
- roundtrip do cache DHT + sobrescrita;
- concorrência: leituras de duas threads em handles clonados;
- banco corrompido: arquivo truncado / lixo → erro tipado, sem
  pânico (`catch_unwind`);
- M6b: snapshot escrito no intervalo, estritamente abaixo do tip;
  boot a partir do snapshot = replay completo bit a bit (state_root
  + SMT, estado lógico completo, tip); snapshot alterado (tip_hash
  corrompido, estado indecodificável, height > tip, meta de
  comprimento errado) → fallback silencioso ao carregamento
  completo, mesma chain final; sufixo reproduzido < intervalo;
  substituição do snapshot (sem acumulação) ao cruzar o intervalo;
  store pré-M6b sem tabelas de snapshot carrega identicamente;
- M8: índice anti-replay reconstruído no boot — uma tx reincluída
  dentro da janela é rejeitada (`TxReplay`) após o restore; uma tx
  mais velha que a janela é admitida (poda inalterada, as regras
  de estado decidem); contagem exata == índice vivo (==
  `load_chain_replay`) para um store mais curto e mais longo que a
  janela; boot por snapshot: a parte da janela abaixo de H é
  reconstruída pelo rescan, a acima pelo replay do sufixo;
- P0.2: índice de nomes — `resolve_name == DomainId::from_name`
  para 120 domínios (vivos, após restart, após wipe + rebuild),
  `resolve_tld` para o TLD reivindicado; o GC de um domínio
  expirado remove a entrada (sem fantasmas), antes e depois do
  rebuild; separação estrita dos namespaces domínio/TLD; um append
  falho não deixa entrada de índice; a chave do índice é a derivação
  BLAKE3 documentada;
- P0.3/P0.4 (`tests/verified_snapshot.rs`): export → import →
  boot == replay completo (estado, tip, `state_root_smt` bit a
  bit; o estado importado é o do bloco finalizado, não o do
  tip); página falsificada num byte → rejeição antes de qualquer
  escrita (store estritamente inalterado); página ausente →
  rejeição; `state_root` errado (manifesto ↔ checkpoint, depois
  raiz recomputada ≠ raiz assinada) → rejeição; contadores /
  alturas / tips inconsistentes / ids duplicados → rejeição; a
  importação exige um store vazio; a exportação exige um snapshot
  persistido na altura do checkpoint; contrato de âncora
  (`append_bootstrap_anchor`: marcador exigido, consistência,
  idempotência); formato serializado estrito (bytes em excesso,
  tags, limites DoS); snapshot em H depois blocos H+1..H+k
  juntados → chain idêntica ao nó completo (estado/tip/raiz), boot
  sem histórico abaixo de H, índice anti-replay = exatamente as txs
  em/acima de H.

[`ChainState`]: ../../../crates/scone-blockchain/src/state.rs
