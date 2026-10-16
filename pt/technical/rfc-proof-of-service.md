# RFC — Prova de operação e comitê de finalidade (proof-of-service)

> Status: NORMATIVO — implementado de ponta a ponta (etapas 1 a 5:
> core/protocol/blockchain/network/CLI; este documento é a
> forma consolidada). Toda regra abaixo é verificada contra o código;
> as constantes citadas são os valores codificados.
> Data: outubro de 2026 (consolidação normativa). Toca:
> `scone-core`, `scone-protocol`, `scone-blockchain`,
> `scone-network`, `scone` (CLI).

## 1. Motivação

O pool de finalidade é o pool de owners (owners de domínios/TLDs
vivos). Possuir concede um assento — **operar** não concede nada.
Esta RFC liga o direito de assinar checkpoints à prova de operação de
um relay online, sem jamais condicionar a propriedade:

- **Liberdade preservada**: um TLD privado permanece atribuível sem
  nenhuma condição de atividade pelo tempo que seu mandato for
  renovado (`RenewTld`, mandato de 3 anos). Apenas a *assinatura de
  checkpoints* é ganha pela presença.
- **Incentivo a operar**: operar um TLD privado (ou qualquer
  assento) = sentar no comitê. O operador participa de manter a
  rede de pé e o protocolo o reconhece.
- **Descentralização mensurável**: o comitê torna-se o conjunto dos
  operadores *efetivamente online*, não chaves dormentes.

## 2. Visão geral

Uma transação `Heartbeat` leve (discriminante `0x71`),
assinada por uma chave de **operador**, emitida uma vez por época.
O estado mantém uma janela deslizante de heartbeats recentes. O
comitê de uma época é eleito **do subconjunto do pool cujo operador
efetivo tem um heartbeat fresco**. Sem heartbeat: sem assinatura,
mas sem perda de direitos de propriedade.

## 3. A transação Heartbeat

```
Heartbeat {
    operator: PublicKey,     // Ed25519 key of the operator (the signer)
    owner: OwnerId,          // identity RE-DERIVED from operator (never trusted)
    tip_seen: [u8; 32],      // last tip seen by the node
    height_seen: u64,        // corresponding height
    timestamp: u64,          // Unix seconds
} + Ed25519 signature over the canonical payload (network injected)
```

Corpo assinado (wire, `scone-protocol`):
`operator[32] ‖ owner[32] ‖ tip_seen[32] ‖ height_seen.v ‖
timestamp.v ‖ operator[32]` (a chave do assinante aparece na cauda
do corpo, como em toda tx) `‖ signature[64]`, ou seja ~230 bytes
codificados — **sem PoW**: o custo é a presença contínua (um
heartbeat por época, por meses).

O assinante É o operador: `owner` é sempre rederivado de
`operator` (nunca tomado da entrada), e a decodificação rejeita com
um erro tipado (`InvalidOwner`) uma chave de assinatura que difere
do campo `operator` da cabeça.

- `tip_seen`/`height_seen` servem à observabilidade (detecção de
  operadores bugados ou atrasados); eles NÃO influenciam a
  validade (um nó levemente atrasado continua um nó vivo).

### 3.1 Validação pura (scone-core)

1. Assinatura Ed25519 válida sobre o payload canônico da rede.
2. `owner` == identidade derivada de `operator` (`InvalidOwner`
   caso contrário) — verificado no encode, no decode E por
   `validate()`.

### 3.2 Regras de estado (aplicação, scone-blockchain)

Mediante a aplicação de um heartbeat num bloco portador de altura
`block_height` (toda rejeição precede qualquer mutação):

1. **Anti-replay entre épocas**: `timestamp` estritamente crescente
   por operador — rejeição tipada `StaleHeartbeat` se
   `timestamp <= operator_last_hb[operator]`.
2. **Unicidade por época**: apenas um heartbeat aceito por
   (`operador`, época do bloco portador) — duplicata rejeitada
   com um erro tipado (`HeartbeatDuplicate`). A época é
   `block_height / epoch_min_blocks` (mesma definição que a
   finalidade, nunca uma constante nova).
3. Inserção jornalizada na janela + atualização de
   `operator_last_hb` (reversível num reorg, undo bit a bit).

### 3.3 Anti-spam

- 1 tx aceita por operador por época — o fluxo máximo é
  `pool_size / epoch_secs`, desprezível.
- Sem taxas (pré-economia): a taxa é estruturalmente limitada.

## 4. Delegação de operação

O vínculo assento → operador passa por duas transações dedicadas:

- `SetTldOperator { tld_id, operator }` (discriminante `0xA9`),
  assinada pelo owner **atual** do TLD **privado** (um TLD aberto
  não tem ninguém para assinar legitimamente — rejeição tipada
  `TldIsOpen`). `operator: None` = revogação, volta à
  auto-operação pela chave do owner; revogável instantaneamente.
- `SetDomainOperator { domain_id, operator }` (discriminante
  `0x5E`), assinada pelo owner atual do domínio. Mesma
  semântica.
- `RegisterTld` / `RegisterDomain` / `AssignDomain` NÃO carregam
  campo `operator`: o vínculo assento → operador é sempre um fato
  on-chain recente, resolvido no tip.

A opção de operador no wire é canônica: `0x00` = `None`
(auto-operação), `0x01` = chave bruta de 32 bytes; qualquer outro
valor é rejeitado no decode.

O estado mantém:

```
seat_operators: HashMap<OwnerId, OperatorKey>   // per seat
operator_seats: HashMap<OperatorKey, u64>       // bounded counter
operator_last_hb: HashMap<OperatorKey, u64>     // last accepted ts
```

Regras de aplicação (`apply_seat_operator`, jornalizadas):

- **Idempotência**: re-delegar ao mesmo operador (ou revogar uma
  delegação inexistente) é um no-op não jornalizado (o undo bit a
  bit o exige).
- **Limite anti-aluguel**: um operador só pode ser delegado por
  **um único assento** por vez (`MAX_SEATS_PER_OPERATOR = 1`). Um
  segundo assento delegando a um operador já ocupado → rejeição
  tipada `OperatorAlreadyDelegated`. Justificativa: sem limite,
  um grande operador vende assinatura no atacado e recria a
  centralização — exatamente o que o comitê eleito deve
  evitar.

## 5. Janela de frescor

### 5.1 Janela deslizante

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- Entrada inserida na aplicação de um heartbeat válido (§3.2).
- Purga determinística a CADA aplicação de bloco
  (`gc_heartbeats_journaled`): toda entrada de época `e` com
  `e + HEARTBEAT_WINDOW_EPOCHS <= epoch` é removida (jornalizada —
  reversível num reorg, como as filas de expiração).
- `HEARTBEAT_WINDOW_EPOCHS = 2`: a época atual + a anterior
  — um operador pode perder UMA época sem perder o frescor,
  a segunda o expira.
- **Teto estrutural de memória**: a janela detém no máximo
  `HEARTBEAT_WINDOW_EPOCHS × operator_count` entradas (uma por
  operador por época, chave de tupla); `operator_last_hb` é
  limitado pelo número de operadores e nunca é purgado (a regra
  anti-replay é monótona).

### 5.2 Frescor

"Fresco" = presença de uma entrada `(e, operador)` para `e` em
`[epoch − HEARTBEAT_WINDOW_EPOCHS + 1, epoch]`. Como o estado
purga a cada bloco, a presença em `recent_heartbeats` É o
frescor (`operator_is_fresh`).

### 5.3 Operador efetivo

O **operador efetivo** de um assento é `seat_operators[seat]` se
uma delegação viva existe, caso contrário a chave do owner
(`effective_operator`). O assento assina sob essa chave efetiva —
uma delegação move o direito de assinar, nunca a propriedade.

## 6. Eleição do comitê

```
eligible = { effective operators of the frozen pool's seats
             (banned key excluded) having a fresh heartbeat }

if |eligible| >= MIN_FINALITY_COMMITTEE_SIZE (= 4) :
    committee = select_committee(seed, eligible)   // existing draw
else :
    committee = select_committee(seed, pool)       // safe degradation
```

Detalhes normativos (verificados em `finality.rs`, especificados em
`/docs/en/technical/blockchain.md` § "Assentos e o comitê de
finalidade"):

- o `pool` é o pool **congelado no último checkpoint finalizado**
  (`FinalizedBase.pool` — nunca o estado vivo): uma
  visão local divergiria entre nós e o quórum nunca se
  formaria;
- a `seed` deriva do hash do conteúdo do checkpoint anterior
  (`signing_hash`);
- todo assento do pool congelado é resolvido ao seu operador
  EFETIVO pelo estado no tip (delegação e heartbeats são
  fatos on-chain recentes: dois nós no mesmo tip computam o mesmo
  conjunto); chaves banidas permanecem excluídas;
- o sorteio roda sobre o conjunto ordenado/deduplicado das chaves
  efetivas (`select_committee` inalterado).

**Degradação explícita e segura**: uma rede em bootstrap (sem
heartbeat), ou uma queda geral dos operadores, retorna ao pool
inteiro — a finalidade nunca é perdida por construção. Esta é a
regra "nunca deadlock" do projeto aplicada aqui.

### 6.1 Bootstrap a frio

Época 0 (e enquanto menos de `MIN_FINALITY_COMMITTEE_SIZE`
operadores estiverem frescos): o pool inteiro. A restrição aperta
naturalmente à medida que os operadores emitem. Nenhum período
transicional especial a configurar.

## 7. Emissão automática (scone-network)

A cada ciclo de produção (`Wake::Produce` — nenhum timer novo), o
relay verifica se uma FRONTEIRA DE ÉPOCA foi cruzada
(`height / epoch_min_blocks`, mesma definição que a finalidade).
No primeiro ciclo de uma nova época, ele emite um heartbeat se:

1. uma chave de âncora está armada (`--anchor-key` ou auto), E
2. essa chave ela mesma detém um assento vivo (checagem LOCAL: a
   chave está no pool de elegibilidade on-chain — owners de
   domínios/TLDs vivos, chaves banidas excluídas).

Anti-spam local: no máximo UMA emissão por época por nó
(memo `last_emitted_epoch`, memória local, nunca estado da
chain); o memo só é tomado após uma injeção bem-sucedida no
mempool — uma falha repete no ciclo seguinte. Nada aqui é fatal:
no pior caso o heartbeat espera o ciclo seguinte.

Um **operador delegado** (que não é ele próprio um owner) não é
coberto pela emissão automática: ele emite via o construtor da CLI
do §8 (operação manual ou air-gapped). O heartbeat NÃO é
necessariamente produzido num bloco pelo próprio emissor: é uma
tx como qualquer outra (mempool, produção por qualquer um).

## 8. CLI (`scone`)

O construtor offline de heartbeat (removido com a superfície
`tx`) foi removido com a superfície tx offline
(cli-simplify-v1). Os relays emitem seu próprio heartbeat
automaticamente (§7); a emissão manual/air-gapped, se um dia
necessária de novo, voltaria como um comando dedicado. Os padrões
eram: `tip_seen` = 32 bytes zero
(apenas observabilidade), `height_seen` = 0, `timestamp` = now.
- Como em toda tx, `tx sign` **re-vincula** operador e owner à
  chave de assinatura REAL (valores de entrada nunca confiados);
  `tx verify` valida localmente.

## 9. Observabilidade

- `NodeStatus` (mensagem wire p2p `0x0d`) carrega
  `fresh_operators: u32` (varint): o número de operadores com
  heartbeat fresco — o subconjunto vivo do pool elegível ao
  comitê (operadores efetivos dos assentos do pool, chaves
  banidas excluídas, ordenados/deduplicados). Preenchido pelo
  relay a partir de `state().fresh_operators(height)`; exibido
  por `scone status` como `fresh operators: N`.
- Argumento de campanha: "operar um TLD privado = sentar no
  comitê de finalidade do DNS descentralizado" — verificável
  on-chain.
- Superfície futura (não especificada): `operator_status` — por
  assento, o operador delegado, frescor (fresco / expirado /
  nunca visto), último `tip_seen`. O RPC local de controle foi
  removido (nó somente p2p); a superfície está ainda a definir.

## 10. O que esta RFC NÃO faz

- **Nenhuma perda de posse** por inatividade (opção A — decaimento
  do TLD não operado — é explicitamente rejeitada).
- **Nenhuma recompensa** (opção C aguarda uma camada econômica).
- O heartbeat não prova a *qualidade* do serviço (blocos
  produzidos, registros DHT servidos) — apenas presença. Contar a
  contribuição real é uma extensão futura possível, mais
  manipulável.

## 11. Invariantes protegidos

- Determinismo bit a bit: a janela e a eleição são funções puras
  do estado aplicado (jornalizadas, undo bit a bit).
- Nunca entrar em pânico sobre dados de rede: heartbeat ou
  delegação malformados → rejeições tipadas (`InvalidOwner`,
  `StaleHeartbeat`, `HeartbeatDuplicate`,
  `OperatorAlreadyDelegated`, `TldIsOpen`, `NotTldOwner`,
  `NotOwner`).
- A posse é incondicional; apenas a assinatura de checkpoints é
  condicional.

## 12. Constantes e parâmetros (valores codificados)

| Nome | Valor | Local |
|---|---|---|
| `HEARTBEAT_WINDOW_EPOCHS` | 2 | `scone-blockchain/src/state.rs` |
| `MAX_SEATS_PER_OPERATOR` | 1 | `scone-blockchain/src/state.rs` |
| `MIN_FINALITY_COMMITTEE_SIZE` | 4 | `scone-core/src/consensus_params.rs` |
| Discriminante `Heartbeat` | `0x71` | `scone-protocol/src/transaction.rs` |
| Discriminante `SetTldOperator` | `0xA9` | idem |
| Discriminante `SetDomainOperator` | `0x5E` | idem |

Estes valores fazem parte do consenso: mudá-los quebra a
reprodutibilidade da chain (janela/purga) ou a eleição
(comitê).

## 13. Plano de implementação — estado final

| Etapa | Crate | Conteúdo | Estado |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`, `SetTldOperator`, `SetDomainOperator`, wire, validação | feito |
| 2 | blockchain | janela deslizante (jornalizada), pool filtrado, eleição, ~15 testes | feito |
| 3 | network | emissão automática, relay de tx | feito |
| 4 | CLI | `tx build heartbeat`, exibição `fresh operators` no `scone status` | feito |
| 5 | docs | esta RFC consolidada em forma normativa, espelho fr/en | feito |
