# Nomenclatura

## Alfabeto

Letras ASCII minúsculas, dígitos e hífens (LDH, RFC 1123):
`[a-z0-9-]`, com o hífen nem na primeira nem na última posição de um
label. Sem maiúsculas, sem underscore.

A validação é **estrita** para ASCII: um nome não canônico é rejeitado,
nunca corrigido. `"ABC"` → erro (sem conversão silenciosa para minúsculas).
Justificativa: uma fronteira de protocolo deve ser determinística; a
canonização implícita cria ambiguidades de identidade.

## Nomes internacionalizados (IDNA)

Nomes não latinos são aceitos como **entrada** e codificados em punycode
(IDNA, RFC 5890/5891, UTS#46): `例子.uip` tem `xn--fsqu00a.uip` como
forma canônica.

- a forma canônica (on-chain, ids BLAKE3, DNS, armazenamento) é sempre a
  **A-label** ASCII: toda regra LDH acima se aplica a essa forma;
- a forma unicode original (**U-label**) é mantida em memória para
  exibição (`DomainName::unicode()` / `TldName::unicode()`);
- o limite de 63 bytes por label e o limite total de 253 bytes aplicam-se à
  forma canônica A-label, não ao UTF-8 bruto;
- ASCII permanece no caminho estrito: IDNA aplica-se apenas à entrada não ASCII
  (`"ABC"` continua inválido);
- política anti-homóglifos (scripts mistos, confusáveis): sem filtragem por
  enquanto, apenas UTS#46.

## TLD

Um TLD é um label DNS como qualquer outro: o limite genérico de label
(RFC 1035: 63 bytes) se aplica, não um limite específico de TLD mais curto.

- padrão: `[a-z0-9-]{1,63}` (LDH, sem hífen inicial/final), sobre a
  forma canônica A-label
- limite codificado: `TldName::MAX_LEN = 63` bytes
- válidos: `uip`, `com`, `test`, `abc12`, `0x`, `a-b`, um TLD de
  exatamente 63 bytes, `éxemple` (→ `xn--exemple-2ua`), `例子`
  (→ `xn--fsqu00a`)
- inválidos: um TLD cuja A-label exceda 63 bytes, `ABC`, `-ab`,
  `ab-`, string vazia

## Domínios

- forma: `label[.label]*.tld` — pelo menos dois labels (`name.tld`);
- cada label: LDH `[a-z0-9-]{1,63}`, hífen nem no início nem no fim;
- comprimento total: ≤ 253 bytes;
- nenhum label vazio (`a..b`, `.a`, `a.` inválidos);
- o último label deve ser um TLD válido.

### Subdomínios

O tipo `DomainName` aceita a hierarquia completa desde o início:

```text
example.uip
shop.example.uip
api.shop.example.uip
```

Um subdomínio tem um `DomainId` **distinto** do seu pai.
As regras de alocação de subdomínios (direitos do owner do pai, etc.) são
trabalho futuro.

## Canonização

Um nome validado já é canônico: `canonical() == as_str()`.
Portanto, rejeitados hoje: maiúsculas, pontuação diferente do separador
`.`, espaços, unicode, caracteres invisíveis, etc.

## DomainId

```text
DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || canonical_name)
```

- determinístico: mesmo nome canônico ⇒ mesmo id, em todo lugar, para sempre;
- injetivo na prática: dois nomes distintos ⇒ dois ids distintos;
- o nome bruto **nunca** deve servir como identificador interno
  (chaves da blockchain, chaves do DHT) — apenas para entrada e exibição;
- o prefixo `SCONE-DOMAIN-V1` isola os domínios de derivação: mudar a
  regra (alfabeto, normalização) equivale a criar um novo
  namespace versionado (um novo prefixo de derivação), sem reescrever o
  existente.

### Representação textual

Codificação hexadecimal em **minúsculas** (`0-9a-f`) dos 32 bytes brutos:
exatamente 64 caracteres, sem prefixo (`0x`), nunca truncado.

```text
DomainId([0xab; 32]) → "ababababababababababababababababababababababababababababababababab"
DomainId([0x00; 32]) → "0000000000000000000000000000000000000000000000000000000000000000"
```

Esta forma é produzida por `Display` (o `Debug` do Rust imprime
`DomainId(<hex>)`, mesma string). Qualquer saída consumida por um humano ou
uma máquina fora do formato binário canônico — logs, mensagens de erro,
saída da CLI — DEVE usar esta representação e nenhuma outra (sem
truncamento, sem prefixo, sem maiúsculas).

## Limites e regras futuras (não decididas)

- nomes internacionalizados (IDN) via punycode;
- comprimento mínimo dos labels de segundo nível;
- expiração/renovação das reivindicações de domínios.

## TLDId (M7a)

Um TLD registrável carrega sua própria identidade, distinta da de
qualquer domínio:

```text
TldId = BLAKE3-256("SCONE-TLD-V1" || tld)
```

- o prefixo `SCONE-TLD-V1` é **distinto** de `SCONE-DOMAIN-V1`:
  os espaços de ids de TLD e de domínio são disjuntos por construção —
  registrar o TLD `uip` (uma reivindicação sobre o namespace `*.uip`) jamais
  pode ocupar a identidade de um domínio, nem o contrário;
- mesmas regras de exibição do `DomainId`: hex minúsculo, 64 caracteres,
  nunca truncado;
- vetor fixado (derivação pinned por teste):
  `TldId("uip") = ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e`.

A transação `RegisterTld` (reivindicação assinada de um TLD, `owner`
sempre recomputado a partir da chave embutida) faz parte do
enum `Transaction` e do formato wire desde M7b: discriminante
`0x93` (constante opaca arbitrária), payload assinado
`SCONE-TX-SIG-V1`, regras completas em `/docs/en/technical/transactions.md`.
O `RegisterDomain` de um domínio, por sua vez, carrega o nome canônico em
texto claro.

## Reserva/governança de TLDs

Aberto pela M7a: `RegisterTld` reivindica um TLD livre em benefício da
identidade derivada da chave assinante. Desde a M8b a governança está
completa: reivindicação com PoW (dificuldade por rede) abrindo um
mandato renovável de 3 anos (`RenewTld`), namespace **fechado** por
padrão (somente assign), abertura explícita (`SetTldOpen`), transferência
(`TransferTld`), abandono (`RevokeTld`, reassumível). TLDs privados
vivem 3 anos (carência de 60 dias, gestão congelada durante a
carência); TLDs abertos carregam um piso de 1 ano mantido pelos seus
domínios; domínios expiram após 1 ano (renovável, carência de 30 dias —
ver `/docs/en/technical/blockchain.md`).

**TLDs raiz do ICANN**: reservados. `RegisterTld` sobre um TLD da
lista IANA fixada (`is_icann_tld`, ~1.440 entradas + RFC 2606) é
rejeitado com um erro tipado (`IcannTldReserved`) — a raiz legada
pertence ao DNS existente. No momento da resolução, esses nomes são
encaminhados aos upstreams configurados (`--dns-upstream`) ou
RECUSADOS (REFUSED) sem upstream. TLDs adicionados pelo ICANN depois
permanecem reivindicáveis (lista fixada: consistência > atualidade).


## Economia do ciclo de vida — quem paga o quê, e quando

O modelo de autorização segue a natureza do recurso: trabalho
(PoW) paga pelo que entra no commons, uma assinatura (PoS) governa
o que pertence a alguém. Resumo normativo:

### TLD privado

- **Nascimento — PoW**: reivindicado por prova de trabalho, dificuldade por
  comprimento (mainnet: 1 caractere ≈ 34 d de computação; piso ≈ 4.3 d com
  10+ caracteres). O reclamante torna-se o owner e detém um **assento PoS**
  (pool de elegibilidade para validadores — heartbeats exigidos para
  permanecer fresco, delegação a operador disponível).
- **Vida — PoS**: a assinatura do owner autoriza tudo no
  namespace: `assign` (conceder um domínio), `set-tld-open`
  (irreversível — **libera o assento**), `transfer`, `revoke`, e
  a renovação dos domínios que vivem sob ele.
- **Mandato — 3 anos**: a reivindicação abre um mandato FIXO de 3 anos
  (`TLD_TERM_SECS`). **Renovação — PoW igual à criação**
  (`RenewTld`, mesma curva de dificuldade por comprimento; desafio
  vinculado à expiração atual — anti-reuso): todo mandato de vida é pago
  em trabalho.
- **Expiração — carência de 60 dias, gestão congelada**: passado o prazo,
  uma carência de 60 dias (`TLD_GRACE_SECS`) mantém a renovação possível,
  mas toda operação de gestão que não a renovação fica congelada —
  notadamente `assign`: um owner expirado perde os direitos de gestão
  até renovar. Domínios já atribuídos abaixo vivem
  sua própria vida (seu próprio `valid_until`) — o TLD congela,
  não seus filhos.
- **Morte — GC**: mandato + carência esgotados sem renovação → o TLD
  desaparece (assento PoS liberado, reassumível por qualquer um com um
  PoW fresco). Um `TransferTld` carrega o mandato restante (sem
  reset).

### TLD aberto (nasce aberto)

- **Nascimento — PoW reduzido** (um terço mais barato que o privado: um recurso
  comum que ninguém governa). **Sem assento PoS**.
- **Piso — 1 ano** (`OPEN_TLD_FLOOR_SECS`): no nascimento (ou na
  abertura via `SetTldOpen`), um piso de 1 ano impede o
  criar-destruir instantâneo de um TLD sem domínio.
- **Vida**: ninguém assina nada — domínios registram-se em
  autoatendimento. Cada domínio vivo abaixo empurra o relógio da morte
  para trás (ele segue a expiração do domínio mais recente, limitado
  pelo piso).
- **Morte — GC** quando nenhum domínio vive mais abaixo e o
  piso/expiração passou; o nome torna-se reassumível.

### Domínio sob um TLD aberto

- **Nascimento — PoW** (curva reduzida vs TLD: mainnet 1 caractere ≈ 8.5 d,
  piso ≈ meio dia).
- **Renovação — PoW idêntica ao nascimento** (mesma curva de
  dificuldade; o desafio é vinculado à expiração atual, portanto
  distinto do registro e de toda renovação anterior):
  todo mandato de vida é
  pago em trabalho, um domínio não pode ser "colecionado" de graça (≤ 3 anos
  com antecedência).
- **Morte**: expiração → carência de 30 d (renovável) → GC → o nome
  torna-se reassumível por **qualquer um** com um PoW fresco.

### Domínio sob um TLD privado

- **Nascimento — PoS do owner do TLD**: somente o owner (ou seu operador
  delegado) concede (`assign`) — a assinatura é a autoridade, sem
  PoW.
- **Renovação — PoS do owner do TLD**: assinada pelo owner do TLD, campo
  `proof` vazio.
- **Morte**: carência depois GC; a reassunção passa novamente pelo `assign`
  do owner.

### Invariantes transversais

- Todo PoW é salgado por **época**, com uma janela de validade
  **para trás** proporcional à dificuldade (uma mineração de várias
  semanas ainda vale mesmo que a época tenha virado).
- Equívoco do assinante PoS é **slashed** (banimento vitalício — M9).
- Toda operação reverifica tudo no momento da aplicação: nada é
  aceito na fé (hashes, provas e assinaturas recomputados).
