# Modelo de segurança — seleção do comitê e anti-grinding

> Documento normativo. Qualquer divergência doc/código = bug.

## Visão geral

O comitê da época N+1 é sorteado a partir da seed
`next_seed(signing_hash(checkpoint N)) = blake3("SCONE-SEED-V1" || signing_hash(checkpoint N))`,
sobre o **pool de elegibilidade congelado no bloco finalizado pelo
checkpoint N**. `signing_hash` é o hash do **conteúdo** do checkpoint
(época, altura, bloco, prev, raiz de estado — conjunto de assinaturas
excluído): a seed é função pura do QUE foi finalizado, nunca
de QUAL subconjunto de quórum a assinou — duas visões detendo subconjuntos
distintos (ambos válidos) do mesmo checkpoint elegem o mesmo comitê.

## Ameaça: grinding da seed do comitê

Um atacante quer influenciar a composição do comitê. Três janelas
de ataque existem; todas as três são estruturalmente fechadas:

### 1. Antes da seed — impossível moer a própria posição

A seed da época N+1 deriva do **hash do checkpoint N**,
assinado por quórum (floor(2n/3)+1 do comitê N — `quorum_for`). O
checkpoint N compromete o
`state_root` de um estado que **fixa o pool** (base PoS = estado exato
no bloco finalizado, `FinalizedBase.pool`). Produzir uma
seed diferente exigiria um checkpoint diferente assinado por quórum
— ou seja, corromper ≥ 1/3 do comitê N, que é a hipótese de falha
excluída pelo modelo BFT.

### 2. Depois da seed — tarde demais para entrar

Qualquer registro (reivindicação de domínio/TLD, aparição de nova
chave de owner) só conta para épocas subsequentes, cujas seeds ainda
não são conhecidas. O pool é congelado no `FinalizedBase` ANTES que
alguém (incluindo os assinantes do checkpoint N) possa computar a seed
de N+1.

### 3. Seleção "melhor-de-m" sem vantagem

A posição de uma chave no sorteio =
`blake3("SCONE-COMMITTEE-V1" || seed || pk)` (uniforme), cujo único
grau de liberdade é `pk`. Um grinder que gera m chaves e
registra apenas a melhor posicionada deve primeiro **conhecer a
seed** — mas nesse ponto o pool já está congelado.
Registrar m candidatos "por precaução" custa m provas de trabalho
(uma por domínio/TLD) e rende exatamente m/|pool| de presença esperada:
**nenhuma vantagem sobre m registros diretos**. A estratégia
adaptativa (ver a seed, depois escolher o que registrar) é
temporalmente impossível.

### 4. Viés do último produtor (ataque de retenção)

O produtor do último bloco antes de um checkpoint poderia reter
seu bloco para influenciar `signing_hash(checkpoint N)` e portanto a
seed. Duas salvaguardas:

- o checkpoint assina um estado na altura H, e o **pool é
  avaliado no timestamp do bloco H** — reter um bloco
  atrasa o checkpoint mas não muda o pool de uma época
  já congelada pelo checkpoint N−1 (os pools se sobrepõem pouco de
  época a época);
- reter custa ao atacante sua própria recompensa de produção e
  afeta apenas a VIVACIDADE (checkpoint atrasado), nunca a
  SEGURANÇA: dois checkpoints finais incompatíveis exigiriam ≥ 1/3 de
  duplas assinaturas, prováveis por `SlashTx` e punidas com banimento.

Conclusão: uma VRF ou commit-reveal nada acrescentaria aqui — a
seed já é função de um evento assinado por quórum que é
imprevisível-adiasável-mas-infalsificável, e o congelamento do pool
remove a única alavanca adaptativa. Não complicar o protocolo
(restrição P1.7: "se necessário" — análise: não necessário).

## Equívoco e slashing

Uma âncora que assina dois checkpoints conflitantes na mesma época
pode ser provada por qualquer um detendo ambas as assinaturas
(`SlashTx`, discriminante `0x15`): banimento do pool PoS
(jornalizado, reversível em reorg). Os domínios/TLDs do acusado não
são confiscados (sanção mínima — decisão M9).

## Propriedades verificadas por testes

- `checkpoint_hash_engages_signatures`: o hash do checkpoint
  cobre as assinaturas (nenhuma adulteração a posteriori);
- `seed_chain_and_recovery`: a cadeia de seeds deriva exatamente,
  recuperação k incluída;
- `small_committee_never_finalizes`: guarda BFT — um comitê
  abaixo de `MIN_FINALITY_COMMITTEE_SIZE` nunca finaliza;
- e2e 4 relays: um checkpoint finaliza e se propaga, mesma época
  em todos os nós.

## Limites conhecidos (documentados, não corrigidos)

- O pool elegível inclui owners de TLDs vivos (reivindicação = PoW
  gasto) — superfície Sybil limitada pelo custo PoW por chave;
- O `state_root` do checkpoint é verificado em RAM apenas quando o
  checkpoint mira o tip; replay retroativo é um caminho
  storage/relay (ver `docs/technical/blockchain.md`).
