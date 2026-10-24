# Arquitetura

## Primeiro, em palavras simples

Cinco conceitos bastam para entender todo o resto:

1. **Um nome** — um endereço na internet, como `mysite.scone`.
2. **Sua frase de 24 palavras** — sua identidade. É a escritura que
   prova que um nome é seu, e assina cada mudança que você faz.
3. **O livro de registros público** (a blockchain) — o livro-razão
   compartilhado e público de quem possui o quê. Ninguém o controla,
   ninguém pode reescrevê-lo.
4. **Mineração** — como você reserva um nome: seu computador faz um
   trabalho, justo e igual para todos. Sem fila, sem intermediário.
5. **Um relay** — um computador que opera a rede mantendo uma cópia
   do livro e respondendo perguntas. Qualquer um pode operar
   um; quanto mais houver, mais forte a rede.

O restante desta página aprofunda — você pode parar aqui e
já entender o projeto.

## Princípio norteador

O Scone separa autoridade de disponibilidade:

- a **blockchain** é a fonte da verdade para a propriedade e a ordem das mudanças;
- o **DHT** é um sistema de distribuição/replicação, ele **não** é uma autoridade;
- o **armazenamento local** guarda o estado do nó, índices e caches.

Um nó deve tratar qualquer dado vindo do DHT como potencialmente
malicioso: ele só é aceito após verificação criptográfica
(assinatura do owner, correspondência com o ownership / sequence /
record_hash da chain).

## Separação de responsabilidades

```text
BLOCKCHAIN
──────────
Ownership (propriedade)
Registration (registro)
Authorization (autorização)
Ordering (ordenação)
Sequence
Record hash
Informações de consenso

DHT
───
Registros DNS
Registros assinados
Replicação
Disponibilidade

ARMAZENAMENTO LOCAL
────────────────────
Estado local
Índices
Registros DHT em cache
Estado da blockchain
```

## Crates e dependências

```text
scone-crypto     ← camada mais baixa: primitivas puras sobre bytes
    ▲
scone-core       ← tipos do protocolo (depende de scone-crypto)
    ▲
scone-protocol  ← formato binário wire/blockchain
    ▲
scone-blockchain← lógica blockchain, estado canônico (em memória)
    ▲               └ scone-topology (engine pura, zero rede/fs/
                     async; alimentada por scone-network) — scone-genesis
                     (binário de ferramenta one-shot para a gênese da mainnet)
scone-storage   ← persistência local: backend redb + snapshots verificados
    ▲
scone-network   ← relay p2p: libp2p/QUIC, mempool, sync, DHT, DNS
    ▲
scone-keystore  ← arquivos de chaves `.sconekey` criptografados (depende de scone-crypto)
    ▲
scone           ← binário / aplicação
```

### Decisão: `scone-core` depende de `scone-crypto`

O `DomainId` deve ser computável desde a fundação
(`BLAKE3-256(prefix || canonical name)`), mas a criptografia concreta
não deve contaminar o core. Resolução:

- `scone-crypto` é a camada mais baixa: funções puramente determinísticas
  sobre bytes (mais o RNG do SO para geração de chaves), zero tipos do core,
  nenhuma dependência de rede/filesystem/async → nenhuma circularidade possível;
- `scone-core` a usa apenas para hashing;
- assinaturas e chaves Ed25519 vivem em `scone-crypto` e operam sobre
  bytes (payloads canônicos produzidos por `scone-protocol`).

### Papel de cada crate

**scone-core** — nomes (`TldName`, `DomainName`), identificadores
(`DomainId`, `OwnerId`, `PublicKeyRef`), dados DNS (`DnsRecord`,
`SignedDnsRecord`), transações (`RegisterDomain`, `UpdateDomain`).
Determinístico e portável. Proibido nesta crate: libp2p, redb,
tokio/async, rede, arquivos, SO, servidor DNS, lógica blockchain.

**scone-crypto** — primitivas criptográficas. Hoje: `hash256`
(BLAKE3), assinaturas e chaves Ed25519 (`SigningKey`,
`PublicKey`, `Signature`, verificação estrita em lote), envelope
AEAD de grupos de registros (XChaCha20-Poly1305). Planejado: árvore
Merkle. Regra: nenhum algoritmo caseiro, apenas dependências auditadas.

**scone-protocol** — formato binário do protocolo: codificação/decodificação
canônica (varint LEB128 mínimo, identificadores de 32 bytes puros)
de transações, registros DNS, blocos (`Block`/`BlockHeader` com um
campo de consenso opaco) e mensagens P2P, com limites de tamanho verificados
antes da alocação, rejeição de formas não canônicas e versionamento
(`PROTOCOL_VERSION`). Nenhuma lógica de consenso, rede ou armazenamento.
Referência de formato: `/docs/en/technical/protocol.md`.

**scone-blockchain** — lógica blockchain e estado canônico:
identificação de transações (`TxId`), árvore Merkle ordenada,
`BlockHash`, gênese determinística, validação de blocos e transações,
regras de aplicação de estado (`DomainState`/`ChainState`, em memória,
acesso direto por `DomainId`), detecção de forks e escolha de ramo
(`fork_choice`: piso de finalidade + chain mais longa + desempate por
menor hash), finalidade por checkpoints, e abstração de consenso
(`trait Consensus`, implementação permissiva por padrão). O
PoW de registro (minerar/verificar, dificuldades por rede) vive em
`scone-core`. Proibido nesta crate: rede, DHT, armazenamento (redb),
servidor DNS. Referência: `/docs/en/technical/blockchain.md`.

**scone-storage** — persistência local (entregue). O backend concreto é
`redb` (`NodeStore`), escondido atrás de um trait mínimo para que
blockchain/DHT/DNS nunca se acoplem diretamente ao redb; outro
backend (KV distribuído) poderia ser plugado. Inclui o backend de
estado KV paginado, o log de blocos, snapshots verificados (manifesto
com hash + páginas) e o caminho de restauração/replay no restart. Referência:
`/docs/en/technical/storage.md`.

**scone-network** — o relay p2p do nó: swarm libp2p/QUIC,
relay de transações e blocos (validação local sistemática), mempool
limitado, sync de blocos limitado (128 blocos/pedido), DHT Kademlia para
`SignedDnsRecord`, servidor DNS UDP+TCP (GeoDNS sempre ativo —
tabela GeoIP embutida, fallback recursivo), o loop de
checkpoint/heartbeat das âncoras (a CLI fala p2p QUIC direto com o
relay, sem RPC local). Alcançabilidade e descoberta: AutoNAT
v2 (veredito alcançável/NATed), circuit-relay v2 + DCUtR para nós
atrás de NAT e UPnP (mapeamento da porta QUIC pelo roteador,
opt-in). Configuração via arquivo `.env` (ver `.env.example`).
Referência: `/docs/en/technical/relay.md`.

**scone-keystore** — armazenamento criptografado das chaves de assinatura Ed25519
(arquivos `.sconekey`: Argon2id + XChaCha20-Poly1305). Esta crate é dona do
filesystem para que `scone-crypto` possa permanecer pura. Referência:
`/docs/en/technical/keystore.md`.

**scone-topology** — engine PURA de otimização da topologia p2p local:
zero rede, zero filesystem, zero async. Ela não conhece nem a libp2p
nem o swarm: `scone-network` a alimenta com observações (RTT de ping,
conexões) e recebe decisões de gerenciamento de vizinhança.

**scone-genesis** — binário de ferramenta one-shot (kickstarter) para o
preparo da gênese da mainnet: manifesto JSON versionado, alocações,
bloco determinístico. A ferramenta produz os artefatos, o repositório
nunca os versiona (recusa escritas dentro de um workspace Rust;
distribuição reservada ao lançamento, ver `blockchain.md`).

**scone** — binário/CLI. Depende de uma crate apenas quando realmente
a usa (hoje: scone-network para o relay, scone-blockchain,
scone-storage, scone-keystore, scone-protocol, scone-core,
scone-crypto, scone-topology).

## O nó

Um nó Scone combina:

1. uma blockchain (validação de consenso, estado de propriedade);
2. roteamento DHT (publicação/recuperação de `SignedDnsRecord`);
3. armazenamento local (estado, índices, caches);
4. um resolvedor / servidor DNS na frente.

### Iniciando um novo nó (bootstrap de estado, P0.3/P0.4)

Um nó novo NÃO reproduz a gênese: ele parte de um estado
finalizado verificado.

```text
checkpoint finalizado (comitê, quórum de assinaturas — validado pela
        │              camada da chain antes de qualquer uso)
        ▼
snapshot verificado (manifesto + páginas paginadas; o importador
        │              RECOMPUTA a raiz SMT a partir das entradas
        │              e a compara com a state_root assinada —
        │              um snapshot falsificado/incompleto é rejeitado
        │              antes de qualquer escrita)
        ▼
bloco âncora H (pedido à rede POR HASH)
        │
        ▼
blocos recentes H+1..H+k (relay, altura crescente)
        │
        ▼
sincronizado — no boot, replay APENAS do sufixo acima de H
```

Dois perfis de nó:

- **nó arquivo** (flag de operador): guarda todos os blocos históricos,
  os serve à rede, inicia pelos caminhos existentes;
- **nó de estado completo**: detém o estado completo em H e o sufixo,
  mas não o histórico abaixo de H — ele valida blocos futuros e
  serve estado, não serve histórico.

Este caminho é OPT-IN: importar um snapshot verificado é uma ação
explícita do operador (`import_verified_snapshot`), nunca automática —
o comportamento padrão de um nó existente não muda.
Detalhes normativos: `/docs/en/technical/storage.md`, seções
"VERIFIED state snapshot" e "State bootstrap".

## Fluxo de resolução DNS

```text
Consulta DNS (example.uip)
    │
    ▼
Nó: DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || "example.uip")
    │
    ├── snapshot DNS autoritativo (RAM) ─► desconhecido/expirado → NXDOMAIN
    │                                          (sem tocar o core)
    ├── cache DNS fragmentado (qname,qtype) ─► hit: entrada compartilhada,
    │                                          verificação preguiçosa
    ├── cache local (SignedDnsRecord) ─────► verificação (assinatura, owner,
    │                                          sequence, record_hash vs chain)
    ├── DHT get(DomainId) ─────────────────► mesma verificação
    │                                          (miss → cache negativo de 30 s)
    ▼
Resposta DNS (se e somente se válido)
```

O snapshot DNS (existência + expiração de cada domínio vivo,
publicado após cada bloco aplicado) garante que uma decisão de
existência nunca passa pelo run loop do relay: o core só é
tocado para buscas reais de registros.

## Evolutividade

- preimages versionadas (`SCONE-DOMAIN-V1`, `SCONE-PUBKEY-V1`,
  `SCONE-OWNER-V1`, …): mudar uma regra de derivação cria um novo
  namespace em vez de quebrar o existente;
- `RecordData` extensível via variantes + `Unknown { type_code, data }`
  para tipos desconhecidos no momento da leitura;
- `PROTOCOL_VERSION` (`scone-protocol`) para quebras do formato wire;
- a chain armazena apenas hashes: dados grandes ficam no DHT.
