# Relay de rede (M4) — arquitetura, fluxos e controle pela CLI

> Especificação normativa do relay de rede do Scone (`scone-network`).
> Qualquer divergência doc/código = bug (corrigir a fonte da verdade).

## Visão geral

O **relay** é o nó P2P completo do Scone: ele mantém a blockchain
canônica (RAM + `NodeStore` redb), retransmite transações e blocos,
produz blocos (modo devnet), serve o protocolo de sync, publica e
resolve `SignedDnsRecord`s num DHT Kademlia, e expõe o
**protocolo p2p** (`/scone/reqres/1`) que a CLI consulta diretamente
(unificação RPC→p2p: o canal TCP JSON sumiu).

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

`crates/scone-network` é a **única crate async** do workspace:
tudo abaixo dela (crypto → core → protocol → blockchain →
storage) permanece síncrono e puro.

## Admissão de conexões (P1.5)

Limites duros no nível do swarm (comportamento `connection_limits`
da libp2p, primeira entrada do `SconeBehaviour`):

- `MAX_TOTAL_CONNECTIONS = 128` — teto de conexões estabelecidas
  (entrantes + saintes) por relay; o excesso entrante é RECUSADO,
  nunca enfileirado;
- `MAX_PER_PEER_CONNECTIONS = 4` — um único par não pode monopolizar
  a cota total abrindo muitas conexões;
- `MAX_PENDING_INCOMING = 32` — guarda slowloris: meias-conexões não
  autenticadas nunca esperam indefinidamente.

Estes limites completam os limites por mensagem existentes
(`MAX_MESSAGE_LEN` 1 MiB) e por orçamento (sync `max ≤ 128`
blocos, DNS TCP 64 conexões/512 B/5 s). Um atacante por inundação de
conexões recebe recusas, não crescimento de memória.

## Descoberta de pares

Três mecanismos complementares, nesta ordem de prioridade:

1. **Seeds explícitos** (`--bootstrap`, multiaddrs completos) — sempre
   têm prioridade; fornecer uma lista não vazia SUBSTITUI todo o
   resto. `--no-builtin-seeds` esvazia os seeds embutidos para
   implantações 100% privadas.
2. **Seeds embutidos** (`builtin_seeds(network)` em
   `scone-network/src/config.rs`) — congelados a cada release oficial
   do binário, por rede. Um relay não configurado entra na rede
   sozinho. (Vazio na testnet enquanto a testnet pública não for
   implantada.)
3. **Diretório aprendido** (`<data-dir>/peers.json`) — todo endereço
   aprendido via `identify` é retido (última vista; teto de **2000
   entradas** — endereços, não conexões, ~400 KiB RAM no
   máximo — e um **corte por idade de 90 dias**: uma entrada vista
   pela última vez há mais de 90 dias é descartada na escrita e
   ignorada no load, mantendo os 2000 slots cheios de pares
   plausíveis em vez de fósseis pré-churn cuja única contribuição
   seria um timeout de discagem no boot) e salvo ~1×/min no tick de
   produção. No boot sem seed e sem bootstrap, o diretório é
   recomposto: o relay encontra seus pares habituais sem
   configuração. É um cache: arquivo ausente/corrompido = diretório
   vazio, nunca um erro.
4. **UPnP** (`--upnp`, opt-in) — mapeamento da porta de escuta QUIC
   pelo roteador doméstico: o nó torna-se alcançável de fora
   SEM circuit relay. No `NewExternalAddr`, o endereço externo é
   registrado como o endereço externo real do swarm
   (`add_external_address` — visto por identify/autonat, candidato a
   transport-relay). Sem gateway IGD (`GatewayNotFound`)
   ou gateway não roteável (`NonRoutableGateway`): um simples log de
   info, o mapeamento é pulado — casos normais numa LAN doméstica. Um
   mapeamento expirado (renovação falha) é um aviso.

O UPnP permanece **desativado por padrão** (`enable_upnp` =
`false` em `config.rs`): uma implantação pública não fala com
nenhum roteador sem consentimento. **A descoberta mDNS foi
removida** (remove-mdns-discovery, 2026-10-22): o peering vem
exclusivamente do bootstrap explícito, dos seeds builtin e do
Kademlia — a flag legada `--mdns`/`SCONE_MDNS` continua aceita sem
efeito (compatibilidade de scripts, mesmo tratamento que `--rpc`).

### Otimização de topologia local (sempre ativa)

O relay otimiza ativamente sua vizinhança usando a engine pura
`scone-topology` (ver os docs da crate). **Sempre ativada**; o único
ajuste exposto é um **perfil** (`--topology-profile`,
env `SCONE_TOPOLOGY_PROFILE`, padrão `default`):

| Perfil | min / alvo / máx | aval. | para quem |
|---|---|---|---|
| `tiny` | 2 / 4 / 6 | 120 s | Raspberry Pi, box doméstico, VPS 1-vCPU |
| `small` | 4 / 8 / 12 | 90 s | servidor pequeno, NAS doméstico |
| `default` | 8 / 16 / 32 | 60 s | servidor padrão (padrão) |
| `high` | 16 / 32 / 64 | 45 s | nó soberano, backbone |
| `very-high` | 32 / 64 / 128 | 30 s | hubs fundadores |
| `devtest` | 2 / 4 / 6 | 5 s | **somente LAB/TESTE** (instável por design) |

```
scone relay --topology-profile tiny
```

Os parâmetros protetores (histerese 0,15, cooldown 600 s, carência
120 s, medição 15 s) são idênticos em todo lugar: guardam a
estabilidade e não têm razão de escalar com o tamanho do nó. Um
nome desconhecido é um erro tipado de partida.

Uma configuração incoerente (`min > alvo > máx`, limiar fora de
`[0,1]` — `NaN` incluído —, intervalo zero) é um **erro tipado de
partida**, nunca comportamento silencioso.

**Visão (`TopologyView`)** — alimentada por eventos do swarm:

- `ping` bem-sucedido → observação de RTT (janela de mediana de 8
  amostras);
- `ping` falho → penalidade de confiabilidade;
- `identify` recebido → endereços (≤ 4/par) + protocolos anunciados
  (máscara kad/reqres/identify/relay);
- conexão estabelecida → estado conectado, direção
  (entrante/sainte), idade, circuit-relay detectado;
- conexão fechada → penalidade + contador de churn SOMENTE se não
  solicitada (uma evacuação voluntária não conta);
- falha de discagem sainte → penalidade de confiabilidade.

A visão é limitada (500 pares; além disso, apenas os pares conectados
são memorizados).

**`kad_required` a partir dos kbuckets** — antes de cada avaliação, o
relay percorre a tabela de roteamento Kademlia: um par é protegido se
está `Connected` num bucket, ou `Disconnected` num bucket SATURADO
(k = 20). Um par protegido NUNCA é escolhido para evacuação.

**Loop de otimização** — no intervalo de avaliação do perfil
(60 s sob `default`; ver a tabela acima):

1. recomputar `kad_required` (kbuckets);
2. avaliação da engine (histerese: carência de 120 s, melhoria mínima
   de escore de +15%, cooldown de evacuação 600 s);
3. decisão aplicada: `AddCandidate` → discar o candidato;
   `Replace` → fechar o par evacuado (UMA VEZ — progressivo, sem
   looping) depois discar o substituto;
4. um snapshot de **métricas** é logado (nível INFO):
   contagens conhecidos/conectados/entrantes/saintes/circuito/protegidos,
   RTT médio/mediano, buckets IP distintos, concentração máxima, churn
   (substituições/desconexões).

O escore de vizinhança NUNCA é uma medida de confiança (ver os
docs de `scone-topology`): nenhuma decisão de consenso deriva dele.

### Alcançabilidade (AutoNAT) e relay de transporte (circuit-relay v2)

Todo relay roda AutoNAT (cliente + servidor): o nó pede a seus pares
que tentem uma discagem de VOLTA para ele; o veredito (`reachable`:
sim = alcançável de fora, não = atrás de NAT, unknown = ainda não
testado) é atualizado a cada 5 min e exposto no `NodeStatus` (campo
`nat_reachable`, varint wire 0/1/2) — a CLI o exibe
(`scone status`). Endereços privados contam
(`only_global_ips = false`): uma malha LAN testa sua alcançabilidade
LAN.

Um relay alcançável ativa o **hop** do circuit-relay v2 (pode
retransmitir tráfego para pares NATed) e o **DCUtR** (hole-punching:
dois nós conectados via circuito tentam uma conexão direta e liberam
o relay). Um relay julgado NATed pede automaticamente uma reserva ao
primeiro par alcançável conhecido e escuta através dele (endereço
`/p2p/<relay>/p2p-circuit`).

Regra de produção: um nó NATed via relay não deveria tornar-se
produtor de blocos (latência/disponibilidade degradadas) — a regra
PoS (assento = domínio vivo) ainda não o proíbe; a documentar como
requisito operacional antes da testnet pública.

## Transporte e protocolos libp2p

| Protocolo | Nome | Papel |
|---|---|---|
| QUIC (quic-v1) | — | Transporte principal (udp) |
| identify | `/scone/id/1` | Descoberta de endereços dos pares (preenche a tabela Kademlia) |
| ping | libp2p padrão | Vivacidade |
| Kademlia | `/scone/kad/1` | DHT: chave = `DomainId` (32 B), valor = codificação canônica do `SignedDnsRecord` |
| request-response | `/scone/reqres/1` | mensagens de aplicação scone-protocol |

### Quadro de aplicação (`/scone/reqres/1`)

Cada quadro é `len: u32 BE || bytes` onde os bytes são a
codificação canônica de uma `Message` de `scone-protocol` (o
codec existente — nenhum segundo formato). O comprimento é limitado
por `limits::MAX_MESSAGE_LEN` **antes** de qualquer leitura ou
alocação: um par anunciando um quadro maior é rejeitado sem custo de
memória.

Mensagens trocadas (uma por round-trip pedido/resposta):

| Mensagem | Direção | Resposta |
|---|---|---|
| `Hello { version }` | handshake | `Hello` (versão local) |
| `Ping` / `Pong` | vivacidade | `Pong` |
| `GetBlock { hash }` | consulta | `Block` (janela RAM) ou `Pong` se desconhecido |
| `GetBlocks { start, max }` | sync | **um** `Block` (altura `start`), ou `Pong` se nenhum |
| `Block (b)` | relay de blocos | `Pong` (ack) |
| `Transaction (tx)` | relay de txs | `Pong` (ack) |
| `GetRecord { domain_id }` | DHT local | `Record` (cache local) ou `Pong` |
| `Record (r)` | publicação | `Pong` se aceito, erro caso contrário |

Regra: **exatamente uma resposta por pedido**; as mensagens de
reconhecimento são `Pong`. As respostas de sync carregam um bloco por
vez — o catch-up itera do lado do solicitante (ver "Sync"). Isto é
deliberadamente limitado: nunca mais que um bloco em voo por par.

## Comportamento do relay

### Partida

0. resolução de rede (`--network testnet|mainnet|devnet`, testnet padrão
   enquanto o projeto está em dev — M8b). O data-dir é **por
   rede** (`~/.scone/<network>/`): um store não vazio contendo uma
   chain de outra rede é recusado (`Corrupted`);
1. abrir/criar o store redb (`<data-dir>/chain.redb`);
2. `load_chain(store, network)`: tip O(1) + estados de domínios
   paginados (janela RAM: gênese + tip);
3. construção do swarm (identidade Ed25519 efêmera por sessão —
   persistir a identidade de rede é uma decisão aberta);
4. compor a lista de boot (ver
   [Descoberta de pares](#descoberta-de-pares)): `--bootstrap`
   explícito (deve terminar com `/p2p/<peer-id>`) OU os seeds
   embutidos da rede OU o diretório aprendido (`peers.json`);
5. discar cada multiaddr e registrá-lo na tabela Kademlia
   (uma discagem morta é ignorada — os outros formam a rede).

### Recebimento de uma transação (submissão P2P ou CLI `submit tx`)

1. decodificação estrita limitada (`MAX_MESSAGE_LEN`);
2. validação criptográfica completa (`validate_transaction`:
   vínculo owner/chave, `verify_strict` sobre o payload canônico);
3. **anti-replay PRIMEIRO** (barato, independente de estado,
   definitivo), depois pré-checagem de estado (espelho barato das
   regras de `push_block` — o PoW, que é caro, só é verificado no
   push): um `RenewDomain` repetido responde `TxReplay`, não um erro
   de PoW mentiroso (sua prova vincula a expiração que a primeira
   aplicação ultrapassou). A separação de
   redes é imposta na etapa 2: o payload de assinatura injeta a rede
   do RELAY, uma tx assinada noutro lugar falha com
   `InvalidSignature` (wire 2026-09: sem mais campo `network`);
   `RegisterDomain` → TLD do nome registrado
   (`UnknownTld` caso contrário, D1/M7c) **e aberto**
   (`TldClosed` caso contrário, M8b) depois nome livre;
   `UpdateDomain` → domínio conhecido, sequência exata
   `current + 1`, owner; `RegisterTld` → TLD livre;
   `TransferTld`/`RevokeTld`/`SetTldOpen` → TLD conhecido +
   assinante owner; `AssignDomain` → owner do TLD + nome livre;
   `RenewTld` → TLD privado conhecido + owner, mandato fixo
   rederivado, pré-checagem de janela de carência; `RenewDomain` →
   domínio conhecido + owner, pré-checagem do PoW de renovação
   (vinculado à expiração atual) na altura seguinte;
4. inserção no mempool (capacidade fixa, dedup por `TxId` —
   reenviar uma tx já no pool é no-op);
5. broadcast aos pares (exceto o remetente).

### Recebimento de um bloco (P2P, sync, ou auto-produzido)

1. `push_block`: validação completa existente (pai = tip canônico,
   altura, raiz Merkle recomputada, toda transação validada depois
   aplicada — atômica por bloco);
2. se aceito: o relay lê o resultado da aplicação
   (`push_block_with_gc` → hash + domínios removidos pelo
   GC determinístico) e persiste via `store_block_with_removals`
   (append atômico de delta: bloco + tip + estados modificados — upserts
   E remoções: domínios expirados, TLDs revogados). Um restart
   portanto jamais ressuscita um registro expirado;
3. reconciliação do mempool (txs incluídas são removidas);
4. broadcast aos pares (exceto o remetente).

Um bloco inválido é rejeitado com um erro tipado; o relay continua
rodando. Nenhum pânico sobre dados de rede. As aplicações pesadas
(Argon2 + commit redb) passam por `tokio::task::block_in_place`
(driver síncrono, aceitação p2p, produção): o runtime move as tasks
prontas para outros workers — aplicar um bloco não trava mais o
serviço DNS nem as métricas.

### Snapshot DNS (publicação)

Após CADA bloco aplicado (aceitação p2p — reorgs incluídos —, driver
de sync, produção) E no boot (um restart não deve NXDOMAINar
nomes vivos até o bloco seguinte), o relay publica o
**snapshot DNS autoritativo** no feed `watch` lido pela
superfície DNS (especificação completa em `dns.md`):

- extensão do tip (o caminho esmagador): delta **O(tocados)**,
  copy-on-write — as transações do bloco nomeiam os domínios
  tocados, mais as remoções do GC determinístico
  (`AppliedBlock.gc_removed_domains`, avaliadas no push);
- reorg / boot: **reconstrução completa** (um passe paginado
  `ChainState::dns_facts`).

### Produção de blocos (M5: produtor assinado)

A cada `produce_interval` segundos (2 s por padrão): se o mempool
não está vazio, um bloco é montado pelo `BlockBuilder`
(`timestamp = now`) **assinado por um produtor autorizado** (payload
`SCONE-BLOCK-V1` em `header.consensus`, ver
`/docs/en/technical/blockchain.md`) e segue exatamente o
caminho de aceitação de um bloco recebido. Um relay sem txs não produz
nada.

Chave de produção: a chave de âncora (`--anchor-key`) quando o relay
está armado com uma — uma âncora é um produtor autorizado —, caso
contrário a chave devnet fixa, válida enquanto o pool de produtores
estiver vazio (bootstrap aberto: gênese). Assim que existir um
domínio/TLD vivo, um bloco assinado pela chave devnet é rejeitado
(`InvalidProducer`).

### Loop de âncora (finalidade, M5)

Com `--anchor-key` (keyfile `.sconekey`, passphrase via
`--anchor-passphrase-env`, padrão `SCONE_ANCHOR_PASS` — o
padrão da biblioteca `scone-network` `SCONE_ANCHOR_PASSPHRASE` nunca é
alcançado através do binário, o clap sempre o sobrepõe):

- **rotação do proponente**: exatamente UM membro designado do
  comitê se autopropõe por época (`committee[epoch % size]`,
  determinístico); os demais membros nunca assinam o próprio tip —
  o conteúdo de um checkpoint depende do tip do proponente, e
  autopropostas sem filtro fragmentavam a época em conteúdos
  rivais que cada guarda rejeita com razão como equivocação
  (testnet, 2026-10-21);
- após cada bloco aceito/produzido, se este nó é membro do
  comitê atual **e** o comitê atinge
  `MIN_FINALITY_COMMITTEE_SIZE`, o relay assina o checkpoint
  proposto via seu `SignerGuard` (`signer-state.dat` no
  data-dir — uma chave = um voto/época, fail-closed à prova de crash) e
  o gossipa;
- **voto ao receber**: uma proposta cujo conteúdo é válido na
  NOSSA chain (o bloco referenciado está presente, no mesmo hash)
  e carrega uma assinatura do comitê recebe a NOSSA assinatura
  também — sem ela, cada âncora assinava apenas o conteúdo do seu
  próprio tip e nenhum quorum nunca se formava;
- assinaturas recebidas (`Message::Checkpoint`) são acumuladas por
  hash de `CheckpointData` (limitadas a 256 entradas); com
  assinaturas de `quorum_for(tamanho do comitê)` membros sobre os
  MESMOS dados → `accept_checkpoint` + broadcast do checkpoint
  finalizado;
- **persistência da finalidade**: a janela de checkpoints e a base
  congelada são escritas no store a cada finalização e
  recarregadas na partida. Checkpoints são estado de gossip
  (nunca carregados por blocos) — uma reinicialização sem essa
  persistência partia de uma base vazia e nunca mais podia
  encadear uma janela servida por pares (as épocas de âncora abaixo
  da janela da rede se perderam);
- **catch-up de finalidade**: um relay cuja base está vazia (boot
  de um nó novo) pede `GetCheckpoints` a um par a cada tick até a
  base ser restaurada;
- um relay sem chave de âncora permanece **passivo**: acumula,
  finaliza e retransmite, mas nunca assina;
- `GetCheckpoints` → a janela `checkpoint_window()` é servida;
- qualquer erro do loop é logado e engolido — nunca fatal;
- o status (`GetStatus`) expõe `checkpoints` e `finalized_epoch`.

### Estacionamento de forks e reorgs (correção sticky-minority)

Um bloco irmão na mesma altura, ou um bloco cujo pai não é o
tip atual (`ParentNotTip`/`UnknownParent`), NUNCA é perdido:
ele é **estacionado** (map limitado, `PARKED_CAP = 2048` blocos, ~KiB
cada — RAM de pior caso limitada; um parking cheio silenciosamente
ignora os recém-chegados). Hashes já oferecidos a `accept_block` são
deduplicados (`seen_blocks`): um bloco perdedor re-gossipado não é
re-avaliado.

A CADA tick `Produce`, `reconsider_parked`:

1. drenagem linear dos blocos estacionados cujo pai tornou-se o tip;
2. se um ramo estacionado é mais forte que o tip atual (regras de
   fork-choice INALTERADAS: piso de finalidade, chain mais longa,
   menor hash), adoção — o sufixo desalojado é ele próprio
   estacionado (OS DOIS lados do fork permanecem avaliáveis).

Isto destrava um nó que adotou o lado perdedor de um fork de
alturas iguais: o bloco vencedor, oferecido uma vez depois recusado,
é periodicamente re-avaliado em vez de esquecido para sempre
(achado chaos-30: dois blocos irmãos a 15 ms de distância).

**Persistência de reorg**: adotar outro ramo não pode passar pelo
append monotônico (`NonMonotonicHeight`). O relay aplica
`rewind_to_height(ponto de fork)` depois
`persist_state_snapshot` (reescrita do estado vivo a partir do export
da chain) depois um re-append monotônico do sufixo vencedor (ver
`storage.md`). Uma falha de persistência é logada e degradada (cura no
boot via replay) — nunca fatal.

## Sync (catch-up)

A cada conexão a um par, o relay pede
`GetBlocks(start = altura_local + 1)`. Cada `Block` recebido que
completa exatamente a altura esperada dispara o pedido da
altura seguinte, até o esgotamento (`Pong` = nada mais) ou
`MAX_SYNC_ROUNDS` (10.000) lotes — um limite absoluto mesmo contra
um par mentiroso. Os blocos recebidos passam pela validação completa:
um par malicioso não consegue fazer nada inválido ser aceito.

### DHT de registros

- **put** (fluxo API/web — o canal hex da CLI foi removido):
  decodificação estrita limitada (`MAX_DHT_CACHE_ENTRY`),
  verificação contra o estado on-chain (owner, sequence,
  `record_hash` recomputado — ver "Verificação de registros"),
  depois `kad.put_record` (armazenamento local +
  replicação aos pares mais próximos, `Quorum::Majority` do
  fator de replicação) e cache redb persistente.
- **get** (`domain resolve`): `kad.get_record`; o waiter fica
  vinculado ao seu `QueryId` e ao `DomainId` pedido — uma resposta
  só resolve a consulta correspondente, e a chave do registro deve ser
  exatamente o domínio pedido (verificado **antes** da verificação
  on-chain).
  O resultado é depois verificado contra a chain antes de ser
  retornado. Teto de pedidos simultâneos: `MAX_DHT_WAITERS` (256);
  além, erro imediato. Uma resolução verificada com sucesso é
  escrita no cache persistente (M5): as leituras locais
  (`domain_info`) então a servem sem nova consulta DHT.
- Kademlia roda em modo servidor, sem TTL de registro (a chain
  governa a validade, não o tempo).

### Verificação de registros (regra "a chain é a autoridade")

Um `SignedDnsRecord` resolvido ou recebido só é aceito/retornado
se:

1. o domínio está registrado on-chain;
2. `record.owner` = owner on-chain atual;
3. `record.sequence` = sequência on-chain atual;
4. `record_hash(record.record)` (BLAKE3 recomputado) = o
   `record_hash` comprometido on-chain.

Caso contrário: rejeição silenciosa do lado P2P, erro explícito do
lado CLI. A assinatura Ed25519 do registro é **verificada no
recebimento** pelo relay (`ResolveLocal`: a chave pública do owner é
conhecida on-chain, a assinatura deve destravar — um registro assinado
por outra chave é rejeitado, `record rejected`); ela é também
transportada para futuros verificadores — o compromisso on-chain
permanece o controle primário de integridade.

## Métricas Prometheus

Um endpoint HTTP (formato texto Prometheus 0.0.4, exposição
feita à mão — nenhuma dependência nova) servido por uma task
dedicada:

- **Desativado por padrão**: nenhum servidor HTTP roda sem `--metrics-addr` (métricas opt-in).
- `--metrics-addr ADDR:PORT`: vincula o endpoint nesse endereço (ex. `127.0.0.1:9101` local, `0.0.0.0:9101` para expor).
- `--metrics-token TOKEN`: muda para o modo **privado** —
  `Authorization: Bearer *** required (comparação em
  tempo constante, `401` caso contrário; o token nunca aparece nos logs
  nem na exposição).
- Uma falha de bind **desativa o endpoint** (aviso não fatal) —
  nunca o nó.

O bind efetivo é anunciado no stdout (contrato legível por máquina,
como `p2p:`): uma linha `metrics: <addr>` assim que
escuta.

**19 contadores** (`counter`, monotônicos — hot paths:
`fetch_add(Relaxed)` num `Arc<Metrics>` compartilhado, nunca uma
entrada de consenso):

| Grupo | Séries |
|---|---|
| Chain | `scone_blocks_applied_total`, `scone_blocks_rejected_total`, `scone_checkpoints_signed_total`, `scone_checkpoint_sigs_received_total` |
| Mempool | `scone_txs_admitted_total`, `scone_txs_rejected_total`, `scone_txs_evicted_grace_total`, `scone_txs_included_total` |
| DNS | `scone_dns_queries_total`, `scone_dns_cache_hits_total`, `scone_dns_cache_misses_total`, `scone_dns_local_chain_total`, `scone_dns_dht_answers_total`, `scone_dns_rejected_unverified_total` |
| DHT | `scone_dht_pulls_issued_total`, `scone_dht_negative_hits_total`, `scone_dht_negative_expired_total`, `scone_dht_negative_capped_total`, `scone_dht_cache_evictions_total` |
| Rede de pares | `scone_peer_connections_established_total`, `scone_peer_connections_closed_total`, `scone_peer_connections_incoming_total`, `scone_peer_dial_failures_total`, `scone_peer_blocks_received_total`, `scone_peer_transactions_received_total` |

**6 gauges** (`scone_height`, `scone_mempool`, `scone_peers`,
`scone_domains`, `scone_finalized_epoch`, `scone_addr_book_entries`)
— os cinco primeiros atualizados a cada 15 s via o canal
`RelayCommand` (mesmo caminho que o `status` da CLI — fonte única de
verdade);
`scone_addr_book_entries` é mantido diretamente pelo registro
(atualizado a cada inserção no address-book, fonte da verdade).

Teste E2E: `metrics_endpoint_public_and_private` (200 +
exposição, 401 sem/com token errado, 200 com Bearer).

## Controle pela CLI (unificação RPC→p2p)

O servidor RPC TCP JSON foi **removido**: a CLI (e a ponte axum do
registrador) falam o protocolo p2p **diretamente** — mesmo wire
(`/scone/reqres/1`), mesmos handlers no lado do relay, zero canal
paralelo. Cada comando abre uma conexão QUIC ao multiaddr do nó
(`--peer`), troca UMA `Message`, fecha.

Mensagens de controle servidas pelo relay:

- `GetStatus` → `NodeStatus { network, nat_reachable, tip,
  height, domain_count, mempool, peers, finalized_epoch,
  checkpoints, fresh_operators }` (exibido por `scone status` —
  `fresh_operators` = operadores com heartbeat fresco, RFC
  proof-of-service);
- `LookupName { name }` → `NameState` (estado on-chain, como
  `lookup`);
- `GetOwnedNames { owner }` → `OwnedNames` (domínios + TLDs vivos
  de um `OwnerId`);
- `Transaction (tx)` → submissão com gossip livre (o relay
  re-broadcasta aos pares);
- `GetRecord { domain_id }` / `Record (r)` — resolução e
  publicação DHT (como `scone domain resolve` no lado CLI);
- rejeição de aplicação → `Message::Error(msg)` (chega ao
  cliente como mensagem legível, não um EOF).

O cliente levanta um swarm libp2p mínimo (request-response +
identify), timeout de chamada de 30 s (20 s de round trip). `--peer`
padrão: `/ip4/127.0.0.1/udp/4001/quic-v1/p2p/12D3KooW…` (nó dev
local; o multiaddr completo com o peer id real é anunciado pelo
relay no stdout pela linha `p2p:`).

O canal interno `RelayCommand` sobrevive para a superfície DNS
embutida (`ResolveLocal`, um round trip em processo, nunca um waiter
DHT — ver `dns.md`).

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

Todo comando de controle fala QUIC diretamente ao multiaddr do
relay (`--peer`, nó local padrão — unificação RPC→p2p, sem
mais canal TCP JSON). Relay ausente → mensagem clara "p2p node
unreachable — is a relay running?", saída não nula. A flag
`--rpc` é mantida mas **obsoleta e ignorada** (o servidor RPC foi
removido; apenas compatibilidade de scripts).

### Configuração de ambiente (`.env`)

Na partida, `scone` carrega (via `dotenvy`, antes do parsing do clap)
um arquivo `.env` do diretório atual: arquivo ausente = no-op
silencioso, nunca um erro. Os valores carregados tornam-se a
**linha de base ambiental** das opções que declaram uma variável
(`env = "…"` em `cli.rs`); **uma flag CLI explícita sempre vence**
sobre o `.env`, e o `.env` vence sobre os padrões embutidos:

```text
explicit CLI flag  >  .env variable  >  built-in default
```

O arquivo `.env.example` na raiz do repositório documenta cada
variável (copiar para `.env` e ajustar; `.env` é **gitignored** —
nunca comitar um `.env` real). Variáveis:

| Variável | Flag espelho | Semântica |
|---|---|---|
| `SCONE_NETWORK` | `--network` | `testnet` (padrão), `mainnet`, `devnet` |
| `SCONE_DATA_DIR` | `--data-dir` | diretório de dados (padrão `~/.scone/<network>/`) |
| `SCONE_LISTEN` | `--listen` | multiaddr de escuta P2P (padrão: todas as interfaces, porta efêmera) |
| `SCONE_BOOTSTRAP` | `--bootstrap` | seeds de bootstrap, **repetível** (uma linha por valor — dotenvy acumula); substitui os seeds embutidos |
| `SCONE_GENESIS` | `--genesis` | arquivo de gênese (nós fundadores) |
| `SCONE_MDNS` | `--mdns` | (obsoleto — descoberta mDNS removida; aceito, ignorado) |
| `SCONE_UPNP` | `--upnp` | mapeamento de porta UPnP (`true` = ativado) |
| `SCONE_DNS_ADDR` | `--dns` | endereço de bind do servidor DNS UDP |
| `SCONE_DNS_UPSTREAM` | `--dns-upstream` | fallbacks recursivos, **repetível** |
| `SCONE_DNS_CACHE_LIMIT` | `--dns-cache-limit` | teto de entradas do cache DNS |
| `SCONE_DNS_CACHE_MAX_MEM` | `--dns-cache-max-mem` | teto de memória do cache DNS (por ex. `64MB`) |
| `SCONE_DNS_MAX_INFLIGHT` | `--dns-max-inflight` | teto de consultas DNS concorrentes |
| `SCONE_METRICS_ADDR` | `--metrics-addr` | metrics endpoint address (`ADDR:PORT`)|
| `SCONE_METRICS_TOKEN` | `--metrics-token` | token bearer do endpoint de métricas |
| `SCONE_TOPOLOGY_PROFILE` | `--topology-profile` | perfil de topologia (`tiny`/`small`/`default`/`high`/`very-high`/`devtest`; padrão `default`) |
| `SCONE_ANCHOR_KEY` | `--anchor-key` | keyfile de âncora (checkpoint + produção) |

`SCONE_ANCHOR_PASS` NÃO é a variável-valor de uma opção: é o NOME
PADRÃO da **variável que contém a passphrase da âncora**
(`--anchor-passphrase-env`, padrão `SCONE_ANCHOR_PASS` no nível da
CLI) — o relay lê a passphrase da variável que esta
flag nomeia.

**Âncora automática (padrão LIGADO)**: sem `--anchor-key`, um keyfile
de âncora é gerado/persistido automaticamente no diretório de dados
(`anchor.sconekey`, passphrase `SCONE_ANCHOR_PASS` se definida,
caso contrário gerada aleatoriamente e escrita em `anchor.pass`) — o
relay torna-se produtor assim que sua chave detém
domínios/TLDs vivos (a elegibilidade PoS nunca é grátis).
`--no-anchor-auto` mantém o relay estritamente passivo. A variável de
ambiente padrão de `--anchor-passphrase-env` é `SCONE_ANCHOR_PASS`.

`--network`: o padrão segue o BUILD (mainnet em binários
mainnet-enabled, testnet caso contrário); `devnet` é a
rede local reiniciável (gênese `scone-devnet` própria, PoW quase zero
0..=2 bits — apagar `~/.scone/devnet/` e relançar reseta a chain).
Define a gênese, as dificuldades PoW e o data-dir padrão
(`~/.scone/<network>/`); uma tx de outra rede é recusada de forma
tipada (`WrongNetwork`).

A superfície hex offline (os subcomandos `tx` e `submit`) sumiu
(remove-tx-surface-v2). A CLI cobre os caminhos do usuário
(`domain register`, `domain renew`, `domains`) E a família M8b
como comandos de primeira classe: `tld open`, `tld transfer`,
`tld revoke`, `domain assign`, `domain transfer`.
As reivindicações `register` mineram seu PoW na dificuldade da rede
(milissegundos na devnet).

**Nenhuma opção geo ou de saúde existe**: a geozone está
**imposta pelo build** (tabela DB-IP Lite embutida — semântica
completa em [`dns.md`](dns.md#geodns-geographic-routing)); o
roteamento geográfico está **sempre ativo** sem opção, e o
resolvedor não faz nenhuma verificação de saúde — ele serve o
conjunto de registros assinado como está (a disponibilidade dos
endpoints é assunto dos operadores dos endpoints). Toda flag
`--geo-*`/`--health-*` é uma recusa clap seca.

### `domain register|update` (M5) — transações assinadas num comando

Estes comandos encadeiam construção → assinatura (keystore) →
submissão → (confirmação) numa única invocação.

- **register**: constrói um `RegisterDomain` assinado (timestamp =
  now, proof vazio na devnet), submete-o, espera a confirmação
  on-chain (sequência ≥ 0, timeout de 30 s). Um nome já registrado é
  recusado no lado cliente E no lado relay.
- **update**: o arquivo de registros é a única fonte da verdade. A
  sequência (on-chain + 1) e o `record_hash` (BLAKE3 canônico dos
  registros do arquivo) são **derivados**, nunca digitados: assinante e
  verificadores não podem divergir. Após a confirmação do
  UpdateDomain, o `SignedDnsRecord` (registros do arquivo + owner +
  assinatura Ed25519 sobre a codificação canônica) é publicado ao DHT
  via `put_record` (que reverifica contra a chain). Um
  domínio não registrado, uma identidade não-owner, ou um erro de
  arquivo falham de forma limpa, sem tocar a chain.

#### Arquivo de registros

Texto, um registro por linha, `#` = comentário, linhas vazias
ignoradas; a codificação canônica ordenada faz a ordem do arquivo não
ter efeito sobre o hash comprometido. O arquivo é o conjunto COMPLETO
(um update substitui, não mescla). Tipos:

```text
A 192.0.2.1
AAAA 2001:db8::1
CNAME www.example.uip
NS ns1.example.uip
MX 10 mail.example.uip
TXT free text (spaces normalized)
```

Erros tipados: tipo desconhecido, IP/nome inválido (com número de
linha), conjunto vazio, duplicata, > `MAX_RECORDS_PER_SET`.

## `--genesis` (fundadores)

`scone relay --genesis genesis.bin` — **apenas nós fundadores**
(campanha kickstarter; o arquivo é produzido pelo binário
`scone-genesis`). Regras normativas:

1. O arquivo é rejeitado **antes da leitura** se exceder 1 MiB,
   depois estritamente decodificado (`decode_complete`: nenhum byte
   em excesso). O hash do bloco 0 é **recomputado**, nunca lido do
   arquivo.
2. Validação semântica: altura 0, `consensus` começando pelo id de
   rede da rede pedida (`--network`), nenhuma transação,
   manifesto de alocação canônico. Qualquer violação = erro tipado na
   partida (nunca um pânico, o daemon não parte).
3. Data-dir **virgem**: a chain nasce sobre a gênese alocada — as
   alocações são aplicadas em RAM **antes de qualquer escrita em
   disco**, depois persistidas (estados semeados + bloco gênese sob
   uma chave meta). O primeiro bloco produzido portanto só pode
   estender a gênese alocada.
4. Data-dir **já produtivo**: a gênese de nascimento do store é o
   `prev_hash` do bloco 1 (autocertificante). Hash de arquivo igual =
   OK silencioso (a opção é **redundante após a primeira partida**);
   diferente = recusa tipada — um nó junta-se a uma rede via sync,
   nunca via `--genesis`.
5. Restart **sem** a opção: um data-dir nascido fundador recupera
   sua gênese alocada via a chave meta (rebind em RAM a cada
   boot).

O pino constante do hash de gênese (anti divergência de rede) é uma
frente de trabalho separada.

### Manifesto do binário `scone-genesis` (kickstarter)

O binário `scone-genesis` (crate `scone-genesis`) prepara o manifesto
`genesis-manifest.json` (formato `ALLOC-V1`) depois produz
`genesis.bin`. Regras normativas:

- `scone-genesis new --network <network>` cria um manifesto vazio;
  `add`/`remove`/`list` gerenciam as alocações, `build`
  produz `genesis.bin` (+ hash), `verify` o revalida.
- `add domain <name>` exige que o **TLD do domínio já esteja
  alocado no manifesto** (rejeição imediata caso contrário — a
  chain o teria recusado na aplicação da gênese); seu
  `valid_until` é **automático**: `born_at + 1 ano` (o
  default_validity do manifesto), nunca digitado à mão.
- `add tld <name>`: **aberto por padrão** (de ninguém,
  autoatendimento, piso de 1 ano, owner = zero no wire — tipo
  `0x03`); `--private` = privado (owner, mandato de gênese
  `born_at + TLD_TERM_SECS` — 3 anos, ciclo de vida padrão:
  `RenewTld`, carência de 60 dias).
- Ordenação canônica por nome (conteúdos iguais → mesmo bloco →
  mesmo hash, seja qual for a ordem de inserção); duplicatas
  recusadas.

## Limites e garantias de memória

| Recurso | Limite |
|---|---|
| Quadro P2P | `MAX_MESSAGE_LEN` (1 MiB) antes do parsing |
| Mempool | 4096 tx (rejeição tipada além); `MAX_PENDING_PER_DOMAIN = 4` txs pendentes por domínio |
| Carência do mempool | `MAX_GRACE_ATTEMPTS = 10` reinserções de uma tx descarregada por razão não definitiva (≈ 7 min a 40 s/ciclo), depois evacuação |
| Estacionamento de forks | `PARKED_CAP = 2048` blocos concorrentes |
| Checkpoints acumulados | `MAX_PENDING_CHECKPOINTS = 256` (evacuação: menor hash de dados) |
| Waiters DHT simultâneos | `MAX_DHT_WAITERS` (256); consulta DHT limitada a 30 s (`DHT_LOOKUP_TIMEOUT`) |
| Blocos por resposta de sync | 1 por round-trip; ≤ 10.000 lotes por par (`MAX_SYNC_ROUNDS`); limite de protocolo `MAX_BLOCKS_PER_REQUEST = 128` no pedido |
| Registro DHT | `MAX_DHT_CACHE_ENTRY` (64 KiB) |
| Cache DHT persistente (disco) | limite duplo LRU: `DHT_CACHE_MAX_ENTRIES` (100.000 entradas) E `DHT_CACHE_MAX_BYTES` (2 GiB) — o que atingir primeiro evacua pela ordem de chave; evacuação do GC na MESMA transação que o bloco (um domínio morto pelo GC da chain nunca deixa entrada órfã) |
| Cache negativo DHT | `DHT_NEGATIVE_TTL_SECS` (30 s), teto FIFO `DHT_NEGATIVE_CAP` (4.096) |
| Chain em RAM | gênese + janela do tip (histórico servido do store) |

Regra anti-loop (H2): uma transação só é re-broadcastada se foi
**realmente inserida** no mempool (primeira vista); uma duplicata
(já no pool ou já na chain) nunca é re-rettransmitida.
Um bloco já canônico é detectado e ignorado antes do
`push_block` (sem re-store, sem re-broadcast).

Decisões abertas: identidade de rede persistente do relay, gossip em
broadcast (hoje: request-response unicast a cada par conhecido).
Resolvidos desde a redação inicial: fork choice real (parking +
reconsider_parked, ver acima), consenso PoW (produtor assinado +
comitê de âncoras), resolvedor DNS (a superfície `--dns`, ver
dns.md), canal de controle (unificação RPC→p2p: sem mais TCP
JSON).

## Desligamento gracioso (SIGTERM / SIGINT)

O daemon `scone relay` (primeiro plano) captura SIGTERM e SIGINT
via `tokio::signal`: no sinal, a future `Relay::run` é
cancelada (`tokio::select!` no binário), o que derruba o Swarm
libp2p e fecha os sockets QUIC. O runtime tokio é então
parado com um desligamento limitado (≤ 2 s) que derruba as tasks
DNS desanexadas (UDP/TCP) e fecha seus sockets — o objetivo é que um
`kill` não deixe socket UDP órfão (caso observado na testnet 60).
Um desligamento por sinal loga `relay shutting down (signal)` (INFO,
stderr) e sai com código de saída **0**: um desligamento pedido não é
uma falha. Se a instalação do handler falhou, o comportamento padrão
do sinal aplica-se (degradação, nunca um pânico).


## Transporte Tor (questão aberta)

QUIC/UDP não atravessa proxies SOCKS puros, portanto os relays não
podem trivialmente rodar sobre Tor hoje. Um transporte Tor libp2p
nativo (relay com endereço onion) é o caminho rastreado para a
frente — ver o guia de resistência à censura do site para o estado
atual e a ponte CNAME `.tor` (nomes legíveis para hidden services,
suportados hoje).
