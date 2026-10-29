# Protocolo

Documento de referência do protocolo Scone: formato binário, codificação
canônica, transações, registros DNS, blocos, mensagens P2P, limites e
compatibilidade de versões. A implementação de referência é a crate
[`scone-protocol`](../../../crates/scone-protocol).

Itens marcados **(provisório)** não são finais.

## Versão do protocolo

- constante: `PROTOCOL_VERSION: u32 = 1` (`scone-protocol`);
- transportada por `Hello.version` (handshake) e `BlockHeader.version`;
- regras de compatibilidade:
  - versão recebida **maior** que a versão local → rejeitar
    (`ProtocolError::UnsupportedVersion`); o nó não adivinha o
    formato de uma versão que não conhece;
  - versão `0` → inválida;
  - existe apenas um conjunto de formatos: os bytes de versão de transação
    (`0x01`) e de bloco devem corresponder **exatamente** aos valores
    locais;
- sem divisão major/minor neste estágio: um único inteiro, incrementado a
  qualquer quebra do formato wire.

## Princípios do formato binário

- binário, nunca JSON para dados do protocolo;
- determinístico e canônico: dois objetos logicamente iguais produzem
  **exatamente os mesmos bytes**;
- parsing sequencial, sem alocação quando possível (borrows de bytes);
- toda entrada de rede é não confiável: comprimentos e contadores são
  verificados **antes** de qualquer alocação; a decodificação nunca entra
  em pânico.

### Endianness

- todos os inteiros são varints LEB128 (grupos de 7 bits, menos significativo
  primeiro): a endianness nativa não se aplica;
- endereços IPv4/IPv6 e identificadores de 32 bytes: arrays de bytes brutos,
  sem prefixo de comprimento;
- não há inteiro largo de tamanho fixo no formato: uma única regra de
  codificação de inteiros.

### Varint (LEB128 mínimo)

Cada byte carrega 7 bits de dados; o bit alto é 1 em todos os bytes,
exceto o último. Um `u64` cabe em 1 a 10 bytes.

| valor | bytes (hex) |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16.384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

A codificação deve ser **mínima**: formas superdimensionadas são rejeitadas
na decodificação (`InvalidVarint`). Esta é a condição necessária para que um
valor tenha apenas uma sequência de bytes válida (codificação canônica).
Rejeições: `80 00` (não mínima), `ff×9 02` (overflow de u64), `ff×10`
(comprida demais).

### Strings e bytes

- `bytes`: comprimento varint + bytes brutos;
- `str`: o mesmo, com decodificação UTF-8 obrigatória;
- todo comprimento é limitado (ver [Limites](#limites)) e verificado antes
  da leitura.

### Identificadores

| Tipo | Representação no wire |
|---|---|
| `DomainId` | 32 bytes brutos |
| `TldId` | 32 bytes brutos |
| `OwnerId` | 32 bytes brutos |
| `PublicKeyRef` | 32 bytes brutos |
| `RecordHash` | 32 bytes brutos |
| `BlockHash` | 32 bytes brutos |
| `MerkleRoot` | 32 bytes brutos |
| `DomainName` | `str` (≤ 253 bytes, revalidado por `scone-core`) |

As derivações (`BLAKE3-256` + prefixos `SCONE-DOMAIN-V1`, etc.) permanecem
definidas por `scone-core` (ver `/docs/en/general/naming.md`).

## Transações

```text
Transaction = disc u8 || version u8 || payload
```

| Discriminante | Tipo |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x2A` | `RegisterTldPow` |
| `0x52` | `UpdateDomain` |
| `0x57` | `RenewTldPow` |
| `0x84` | `RegisterDomainPow` |
| `0x93` | `RegisterTld` |
| `0xC3` | `RenewDomainPow` |

Cada discriminante é uma constante opaca arbitrária (sem sequência
lógica; `0x00` nunca válido) — tabela normativa completa em
`/docs/en/technical/transactions.md`.

O byte de `version` (valor atual `0x01`) segue o discriminante:
qualquer outro valor é explicitamente rejeitado
(`UnsupportedVersion(value)`). A maioria das transações é assinada
(Ed25519) — EXCETO as quatro formas sem assinatura pow-to-owner
(`0x2A`, `0x57`, `0x84`, `0xC3`): a prova minerada É a
autorização, e seu desafio liga a chave do beneficiário e a rede.
Especificação completa, payload assinado e regras de validação em
**`/docs/en/technical/transactions.md`** (normativo).

Desde a mudança de wire de 2026-09, **as transações não carregam mais um
campo `network`**: o `network_id` (`str` ≤ 16) é injetado no
**payload de assinatura** (formas assinadas) ou no **desafio PoW**
(formas pow-to-owner) — uma tx feita para uma rede nunca se
transporta para outra (ver `/docs/en/technical/blockchain.md`).

### REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

`RegisterDomain` carrega o **nome canônico em texto claro**, seguido
do seu `DomainId` — a consistência `domain_id == DomainId(name)` é
reverificada na decodificação (recomputada, nunca confiada). `proof` carrega
o PoW de registro (20 bytes: nonce + época + dificuldade — sal de
época `epoch = height/144` do bloco aplicador, verificado ESTRITAMENTE na
aplicação contra a constante da rede e a época do bloco; rejeição
tipada `ProofEpochMismatch` em caso de divergência — M8b, ver
`transactions.md`). `public_key` é a chave Ed25519 do assinante (exatamente
32 bytes); `owner` deve ser a derivação BLAKE3 dessa chave
(recomputado, nunca confiada); `signature` tem exatamente 64 bytes.

### REGISTER_TLD (0x93)

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

Reivindicação de um TLD (registro separado do registro de domínios, M7): o
TLD é **carregado em texto claro** como o nome de um domínio (M8b — o
desafio PoW deriva do nome), e `tld_id` é a derivação `SCONE-TLD-V1`
desse nome, disjunta de todo `DomainId` por
construção. Regras de estado (M8b): TLD livre + não reservado ao ICANN
(`IcannTldReserved` — TLDs raiz do ICANN não são reivindicáveis por ninguém) + PoW
na dificuldade **por comprimento** do label (curva `pow_length` —
ver `blockchain.md`); reivindicado é fechado (somente assign).

### UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

Inváriante reverificado na decodificação: `sequence > 0` e `owner` ==
derivação de `public_key`. Sem `timestamp`: a ordenação passa pelo
`sequence`. A transação carrega apenas referências compactas — o
conteúdo DNS completo fica no DHT (ver abaixo).

## Registros DNS (lado DHT)

Registros DNS completos nunca entram na blockchain: são trocados via
DHT (mensagens `GetRecord`/`Record`) e comprometidos on-chain apenas
pelo seu `RecordHash`.

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

Os códigos de tipo são os **códigos DNS reais** (IANA). O TTL (segundos,
varint) segue o código de tipo e cobre todo o registro: limites
`MIN_TTL = 1` .. `MAX_TTL = 2.592.000` (30 dias; padrão
`DEFAULT_TTL = 86.400`).

| Código | Tipo | Payload |
|---|---|---|
| 1 | A | 4 bytes |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`, `name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16 bytes |
| 33 | SRV | `priority.v`, `weight.v`, `port.v`, `target(name)` |
| 257 | CAA | `flags u8`, `tag` (1–15 B), `value` (`str` ≤ 4096) |
| 32768 | GEOROUTE | GeoRoute (ver abaixo) |
| outro | Unknown | code.v + `bytes` ≤ 4096 |

Exemplo: `A 192.0.2.1` (TTL 86.400) → `01 80 a3 05 c0 00 02 01`
(tipo, TTL varint, endereço).

Um `Unknown` carregando um código conhecido é rejeitado na codificação (a
decodificação nunca pode produzir um).

#### GEOROUTE (GeoDNS)

Tipo privado Scone (faixa não-IANA). Uma regra de roteamento geográfico:

```text
GeoRoute  = scope || count.v(≤ 32) || endpoint…
scope     = 0x00                    -- mundo
          | 0x01 continent.v        -- ONU M49
          | 0x02 country.v          -- ISO 3166-1 numérico
          | 0x03 region.v           -- código de região
          | 0x04 asn.v              -- número de AS
          | 0x05 prefix.addr[16] || prefix.len u8   -- prefixo IP (normalizado)
endpoint  = target || priority.v || weight.v
target    = 0x00 octets[4]          -- IPv4 binário
          | 0x01 octets[16]         -- IPv6 binário
```

**Regra canônica**: os endpoints de um dado escopo são codificados
**ordenados** (prioridade crescente, depois peso decrescente, depois alvo
crescente); a ordenação é aplicada na codificação e um wire não ordenado é
rejeitado na decodificação (`NonCanonical`), como nos conjuntos de
registros. Duplicatas completas de endpoint são rejeitadas.

**Validação**: `priority > 0`, `weight ≤ 100` (0 = endpoint
inativo), ≤ 32 endpoints por escopo (`GEO_MAX_ENDPOINTS`). Um
código de país/continente desconhecido da tabela local
(`continent_of_country`) é **aceito** no nível do wire: a tabela do
resolvedor pode estar mais atualizada que a do publicador. O TTL
é o do `RecordData` pai (inalterado).

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**Regra canônica**: um conjunto de registros é uma coleção não ordenada. Ele
é codificado **ordenado pela ordem lexicográfica da codificação individual
de cada registro**, estritamente crescente (duplicatas rejeitadas). Um wire
não ordenado é rejeitado na decodificação (`NonCanonical`). A ordem do
`Vec` original não é transportada.

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

O hash cobre **apenas o conteúdo do registro**; a assinatura do owner é
verificada separadamente sobre os mesmos bytes canônicos. *(Decisão: o
compromisso não cobre a assinatura — uma versão anterior deste
documento mencionava o "SignedDnsRecord completo"; acoplar o hash à
assinatura não trazia nada e amarrava o compromisso ao esquema de
assinatura.)*

## Blocos

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65.536)
```

- `version`: versão do protocolo (regras acima);
- `height`: altura (gênese = 0);
- `prev_hash`: hash do bloco anterior (zeros para o bloco
  gênese);
- `tx_root`: compromisso sobre a lista de transações (árvore Merkle definida
  por `scone-blockchain`, ver `/docs/en/technical/blockchain.md`);
- `timestamp`: informação de ordenação (Unix, segundos);
- `consensus`: payload do produtor (`SCONE-BLOCK-V1 ‖ pk[32] ‖
  sig[64]`, ver `blockchain.md`) — a gênese de uma rede carrega o
  `network_id` bruto, e uma gênese de alocação (kickstarter da mainnet) pode
  carregar um manifesto `ALLOC-V1` (≤ 60 KiB); limitado por
  `MAX_CONSENSUS_LEN` (64 KiB).

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

Parsing sequencial; o contador é limitado **antes** da alocação; qualquer
truncamento é rejeitado. Diferentemente dos conjuntos de registros, a ordem
das transações é **significativa** (ordem de consenso) e preservada
exatamente.

O `BlockHash` (hash do header canônico) e o `tx_root` são computados
por `scone-blockchain` sobre as codificações definidas aqui (fórmulas
exatas em `/docs/en/technical/blockchain.md`).

## Mensagens P2P

```text
Message = disc u8 || payload
```

O transporte (framing, libp2p…) está fora do escopo; um quadro de transporte não
deve exceder `MAX_MESSAGE_LEN` (1 MiB).

| Disc | Mensagem | Direção | Payload | Limite | Resposta esperada |
|---|---|---|---|---|---|
| `0x01` | `Hello` | ambos | `version.v` | — | `Hello` |
| `0x02` | `Ping` | ambos | — | — | `Pong` |
| `0x03` | `Pong` | ambos | — | — | — |
| `0x04` | `GetBlock` | ambos | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | ambos | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | ambos | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | ambos | `Transaction` | — | — |
| `0x08` | `GetRecord` | ambos | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | ambos | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | ambos | `Checkpoint` | assinantes ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | ambos | — | — | janela de checkpoints |
| `0x0c` | `GetStatus` | ambos | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | ambos | estado do nó (rede, tip, altura, NAT, domínios, mempool, pares, finalidade, operadores frescos) | — | — |
| `0x0e` | `LookupName` | ambos | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | ambos | estado on-chain do nome + TLD, incl. a geração CAS (contadores de mutações, registered_at) | — | — |
| `0x10` | `GetOwnedNames` | ambos | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | ambos | domínios + TLDs vivos do owner | — | — |
| `0x12` | `Error` | resposta | mensagem de erro de nível de aplicação (`str`) | — | — |

### Checkpoint (M5)

Um checkpoint agregado vai ao wire como: tag `SCONE-CKPT-V1` ||
corpo `CheckpointData` (116 B, LE — idêntico ao final da preimage de
assinatura: `epoch(8) height(8) block_hash(32) prev_checkpoint_hash(32)
state_root(32) recovery(4)`) || `count.v` || `pk[32] ‖ sig[64]` por
assinante, **ordenado por pk crescente, sem duplicatas**, `count ≤ 64`
(`MAX_CHECKPOINT_SIGNERS`). Decodificação estrita: truncamento, tag errada,
pk não canônico, duplicatas ou desordenado = rejeição (`None`), nunca
pânico (fuzz de corrupção testado).

`GetCheckpoints` pede a janela de finalidade; a resposta é a
janela `checkpoint_window()` (no máximo `CHECKPOINT_KEEP` checkpoints,
do mais antigo ao mais recente) via uma ou mais mensagens
`Checkpoint`.

O `TxId` é agora definido por `scone-blockchain`
(`BLAKE3-256("SCONE-TX-V1" || canonical(Transaction))`, ver
`/docs/en/technical/blockchain.md`). O mempool e o anti-replay
em janela (`TxReplay`, `REPLAY_WINDOW_BLOCKS = 256`) são entregues no
relay (M4); apenas `GetTransaction` permanece adiado (sua
definição depende das decisões de consenso restantes).

## Divisão blockchain / DHT

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (completo, assinado)
sequence
record_hash
```

A chain diz *quem* possui e *qual é o estado atual* (sequence +
record_hash). O DHT fornece *o conteúdo*. Um registro DHT é aceito
se e somente se:

1. sua assinatura é válida para o owner registrado na chain;
2. seu `sequence` é consistente com (ou superior ao) da chain;
3. o `RecordHash` recomputado sobre sua codificação canônica corresponde ao
   `record_hash` da chain.

## Limites

| Constante | Valor | Papel |
|---|---|---|
| `MAX_NAME_LEN` | 253 | comprimento de um nome |
| `MAX_TLD_LEN` | 63 | comprimento de um TLD |
| `MAX_TXT_LEN` | 4096 | string TXT |
| `MAX_PROOF_LEN` | 256 | prova de registro |
| `MAX_SIGNATURE_LEN` | 128 | assinatura (Ed25519 = 64) |
| `MAX_RECORDS_PER_SET` | 256 | registros por conjunto |
| `MAX_DHT_RECORD_LEN` | 64 KiB | registro DNS assinado codificado (payload DHT: wire, armazenamento, gossip) — limite total verificado na codificação E antes de qualquer decodificação; 15–30× acima de qualquer conjunto realista (SPF+DKIM+DMARC ≈ 2–5 KiB), 16× abaixo do teto de transporte. |
| `GEO_MAX_ENDPOINTS` | 32 | endpoints por escopo GeoRoute |
| `MAX_UNKNOWN_DATA` | 4096 | dados de um tipo desconhecido |
| `MAX_TXS_PER_BLOCK` | 4096 | transações por bloco |
| `MAX_CONSENSUS_LEN` | 64 KiB | payload de consenso de um header (alargado para o manifesto de gênese `ALLOC-V1` — kickstarter da mainnet) |
| `MAX_BLOCKS_PER_REQUEST` | 128 | blocos por `GetBlocks` |
| `MAX_MESSAGE_LEN` | 1 MiB | quadro de transporte (recomendado) |

Qualquer excesso → `LimitExceeded`, detectado antes da alocação: um par
malicioso não pode provocar uma alocação desproporcional na decodificação.

## Codificação canônica

Regra fundamental:

```text
objeto canônico → codificação binária canônica → hash / assinatura
```

- apenas varints mínimos;
- conjuntos de registros ordenados estritamente crescentes;
- um objeto logicamente igual tem exatamente **uma** codificação válida;
- a decodificação rejeita formas não canônicas (`NonCanonical`), portanto não
  existem múltiplas codificações válidas para o mesmo objeto.

## Tratamento de erros

`ProtocolError` (`scone-protocol::error`):

| Variante | Significado |
|---|---|
| `Truncated` | entrada terminou no meio de um valor |
| `InvalidVarint` | varint não mínimo, com overflow ou longo demais |
| `IntegerOutOfRange` | inteiro não cabe no tipo alvo |
| `UnsupportedVersion` | versão zero ou superior à local |
| `UnknownDiscriminant` | discriminante de transação/mensagem desconhecido |
| `LimitExceeded` | comprimento ou contagem além de um limite |
| `InvalidUtf8` | string não UTF-8 |
| `TrailingBytes` | bytes restantes após um valor completo |
| `Validation` | violação de um inváriante de `scone-core` |
| `NonCanonical` | codificação válida por campo mas não canônica |

A decodificação nunca entra em pânico sobre dados de rede.

## Assinaturas

- esquema: **Ed25519** (RFC 8032) via `scone-crypto`
  (`ed25519-dalek`, verificação `verify_strict` — assinaturas
  maleáveis e chaves fracas rejeitadas);
- **transações da chain**: todo `RegisterDomain` / `RegisterTld` /
  `UpdateDomain` embute `public_key` (32 B) + `signature` (64 B) sobre
  o payload `"SCONE-TX-SIG-V1" || canonical_encode(tx_without_signature)`;
  especificação normativa completa: `/docs/en/technical/transactions.md`;
- **registros DHT** (`SignedDnsRecord`): o payload assinado é a
  codificação canônica do `DnsRecord`; a verificação cruza
  `record + owner + signature` com a identidade registrada na
  blockchain;
- regra transversal: `owner` e todos os hashes/compromissos são
  **recomputados**, nunca tomados de um valor fornecido.

## Regras de consenso

Entregues (descritas em `/docs/en/technical/blockchain.md`): produtor de
bloco assinado (payload `SCONE-BLOCK-V1`, comitê eleito por
checkpoint), finalidade por checkpoints agregados (`SCONE-CKPT-V1`,
fork choice no piso de finalidade), PoW de registro por comprimento do
nome.
Tópicos restantes: taxas anti-spam finas, rotação de chave do owner.
