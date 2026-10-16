# Metodologia de auditoria de segurança — Scone (pré-mainnet)

Documento de trabalho. Alvo: confiança para implantação em produção.
Referências entre crates: as regras duras do AGENTS.md (nenhuma cripto
caseira, decodificação estrita, hashes recomputados, determinismo) são a
especificação de requisitos da auditoria.

## 0. Escopo e ameaças (antes de reler qualquer linha de código)

Modelo de adversário: um nó p2p arbitrariamente hostil (pode enviar
quaisquer bytes por QUIC/UDP), um submetente de tx hostil, um operador de
relay curioso, um data-dir roubado. EXCLUÍDOS por design: o cliente final
(DNS é público por natureza), disponibilidade global (o caos já é
testado).

Tabela de ativos, por ordem de criticalidade:
1. **Propriedade** (chain de estado: Register/Update/Transfer/Assign) —
   um falso positivo = roubo de domínio.
2. **Posse de assinatura/chave** (keystore, Ed25519, derivação de
   OwnerId).
3. **Validação de entrada não confiável** (decodificação wire, limites,
   panics).
4. **DoS de recursos** (cache DNS, mempool, DHT, sync).
5. **Determinismo/consenso** (reorg, GC, ressurreição de estado).

## 1. Auditoria estática automatizada (dia 1 — mecânica)

- `cargo audit` (CVEs de dependências: libp2p/quinn/rustls/redb/ed25519).
- `cargo geiger` + revisão dos `unsafe` (esperado ≈ 0).
- `cargo clippy --workspace --all-targets -- -D warnings`
  (já verde, manter como barreira de CI).
- Fuzzing das fronteiras de decodificação (dias 2-3):
  `cargo-fuzz` sobre `scone_protocol::decode_complete` (Message,
  Transaction, Block, SignedDnsRecord, Checkpoint) — esta é A
  superfície onde "nunca entrar em pânico sobre dados não confiáveis"
  deve valer.
  Orçamento: 24 h de fuzzing contínuo, corpus semeado dos testes
  existentes. Critério de saída: zero crashes, zero timeouts, cobertura
  dos ramos de decodificação > 95%.
- Fuzzing do parser wire DNS (dns.rs `parse_query`,
  `extract_answers`) e do parser de arquivo de registros (domain.rs).

## 2. Revisão manual direcionada por componente (dias 4-8)

Ordenada por criticalidade dos ativos, 1 componente = 1 checklist:

### 2.1 scone-crypto + derivação de identidade (dia 4)
- OwnerId = BLAKE3(pubkey): verificar a canonização de bytes (nenhuma
  representação ambígua → 2 chaves, 1 owner).
- Ed25519: DESSERIALIZAÇÃO estrita (chave não canônica rejeitada?),
  verificação com separação de contexto (cada uso tem sua tag de
  domínio: SCONE-TX-SIG-V1, SCONE-BLOCK-V1, registro DHT — nenhum
  replay entre protocolos possível).
- Keystore: passphrase → chave (KDF? iterações?), zeroização,
  permissões de arquivo, oráculo de padding no ciphertext.

### 2.2 Aplicação de transações (dia 5) — o núcleo
Para CADA variante de Transaction (13 hoje):
- A tx é idempotente sob replay (txid único, nonce/sequence)?
- A verificação de propriedade precede qualquer efeito? (assinante ==
  owner aprendido on-chain, nunca confiado da própria tx.)
- Sequência do UpdateDomain: exatamente atual+1 (não ≥) — verificar
  rejeição de lacunas e rejeição de replays.
- Renew: limites de tempo (≤ 3 anos, > agora) — o tempo vem do
  BLOCO, nunca do relógio local (regra já estabelecida, verificar cada
  call site).
- AssignDomain/TransferTld: a pubkey do destinatário é APRENDIDA
  (a armadilha do pool sem pubkey — cf. skill) em todos os caminhos.
- GC/expiração: nenhuma ressurreição possível (o paradoxo
  precheck-OK/lookup-OK/push-fails) — verificar
  gc_expired_journaled.

### 2.3 Camada de rede (dia 6)
- Todo byte remoto passa por um decodificador estrito limitado ANTES da
  alocação (MAX_* de limits.rs) — procurar `Vec::with_capacity(input_len)`
  sem limites.
- requery/sync GetBlocks: limites por par e por ciclo
  (já: 128 blocos/req) — verificar ausência de loops de
  re-download.
- Gossip Record: reverificação contra a chain ANTES do cache/gossip
  (verificado pelos meus testes record_dht) — verificar também o teto de
  fanout.
- kad DHT: um par hostil pode forçar armazenamento de N registros (cota
  por par?) ou evacuação mirada de registros honestos?

### 2.4 Superfície DNS (dia 7)
- Cache: injeção impossível (chave canônica (qname,qtype) —
  nenhuma colisão de caso/codificação), orçamento duro de memória
  (testado), nenhum cache poisoning pelo caminho de fallback (TXID +
  eco da pergunta — verificar que o upstream não pode ser spoofado
  remotamente).
- Respostas: AA honesto, TC honesto, sem amplificação (razão
  resposta/pedido limitada — MAX_INFLIGHT_UDP já presente, verificar
  a replicação TCP).

### 2.5 Storage e gênese (dia 8)
- redb: corrupção de disco → erro tipado, sem pânico (o teste diskfail
  existe no lado do signer — o equivalente no relay?).
- Gênese: o hash cobre o manifesto inteiro; duas gêneses
  diferentes = recusa do data-dir (testado); verificar a impossibilidade
  de injetar uma gênese DEPOIS do fato num data-dir sincronizado.

## 3. Testes adversariais (dias 9-12) — escrever o que falta

Para cada achado da revisão manual, um teste que o prova:
- nó hostil na sim (SimNet com um nó enviando mensagens malformadas/
  discordantes) — estender o SimNet com um papel "byzantino
  passivo";
- malha real de 30 nós com 3 nós hostis (bytes aleatórios, rajadas de
  txs inválidas, registros DHT forjados) — a bateria de caos existe,
  acrescentar o papel de atacante;
- contraprovas: duas implementações da decodificação (o doc
  protocol.md é NORMATIVO — um auditor implementa o wire a partir
  do doc SOZINHO e compara bytes; qualquer divergência = bug do doc OU
  do código).

## 4. Auditoria externa (se houver orçamento) — após 1-3

Escopo mínimo lucrativo para um fornecedor: scone-crypto,
scone-protocol (wire), aplicação de txs (state.rs), ~8-10 k LOC.
Formato: 2 semanas, 1 auditor sênior de cripto + 1 revisor. Entregável:
relatório com severidade, PoCs, correções revisadas. Apoiar-se na
documentação normativa existente (docs/technical) como base da
especificação de requisitos — esta é uma vantagem de custo real.

## 5. Critérios de saída (go/no-go da mainnet)

- [ ] cargo-fuzz 24 h sem crash nos 5 decodificadores
- [ ] Revisão 2.2 completa: cada variante de tx tem sua ficha
      (propriedade/replay/tempo/efeitos) validada pelo revisor
- [ ] Malha de 30 nós incluindo hostis: 48 h sem divergência de tip
      nem corrupção de cache
- [ ] Divergência doc/código: zero (verificada pela implementação
      dupla do wire)
- [ ] CVEs bloqueantes: zero (cargo audit)
- [ ] Auditoria externa do escopo crypto/wire/tx feita (se
      orçada)

## Ordem de execução recomendada

1→2.1→2.2 (o núcleo, 60% do risco)→2.3→2.4→2.5→3 em paralelo
com cada revisão (o teste adversarial é escrito no dia do
achado, não depois).
