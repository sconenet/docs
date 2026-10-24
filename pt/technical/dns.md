# Servidor DNS UDP + TCP (M6 + M7b) — `scone dns`

> Especificação normativa da superfície DNS do Scone.
> Qualquer divergência doc/código = bug (corrigir a fonte da verdade).

## Visão geral

O **servidor DNS** é uma superfície opcional do relay: ele responde
consultas DNS **UDP e TCP** (RFC 7766, mesma porta) para nomes Scone
**apenas a partir de dados verificados contra a chain**, e oferece um
**fallback recursivo opcional** (`addr:port`) para todo o resto.
Ele não detém nem chain, nem store, nem DHT: decisões de
existência/expiração passam pelo **snapshot DNS autoritativo** (uma
visão RAM publicada após cada bloco aplicado — ver abaixo), e as buscas
de registros passam pelo relay (processo), via canal interno
`RelayCommand`/`ResolveLocal` (em processo, um round
trip) — o mesmo caminho verificado que a CLI.

```text
              UDP+TCP :5353 (default 127.0.0.1)
                    │
             scone relay --dns 127.0.0.1:5353
                    │  ┌────────────────────────────────┐
                    ├──┤ Sharded cache, 16 ways (qname)  │
                    │  └────────────────────────────────┘
                    │  Nome com forma Scone?
                    │  ├─ SIM → snapshot DNS autoritativo:
                    │  │        desconhecido/expirado → NXDOMAIN (RAM,
                    │  │        zero round trips ao core)
                    │  │        → busca de apex por sufixo
                    │  │        → `resolve_local` interno (relay)
                    │  │           chain + registro verificado (hash,
                    │  │           owner, sequence) → respostas
                    │  │        apex sem registro publicado → NODATA
                    │  └─ NÃO → fallback upstream (se configurado)
                    │           caso contrário REFUSED
                    │
                    │  orçamento de resposta:
                    │    UDP → 512 bytes (sem EDNS); overflow =
                    │          corte no último registro que couber + TC=1 →
                    │          o cliente repete em TCP
                    │    TCP → 64 KiB (prefixo de comprimento 2 bytes,
                    │          RFC 1035 §4.2.2): conjunto completo
```

## Partida

```bash
scone relay --dns 127.0.0.1:5353            # somente superfície DNS
scone relay --dns 0.0.0.0:53 \
            --dns-upstream 1.1.1.1:53 \
            --dns-upstream 8.8.8.8:53       # + fallback recursivo
```

O relay anuncia o endereço efetivo no stdout
(contrato legível por máquina, como `p2p:`): uma linha `dns: <addr>`
assim que o socket está vinculado.

Cliente de validação: `dig <name> --dns 127.0.0.1:5353
--qtype A` (A, AAAA, TXT, MX, NS, CNAME, ANY).

## Caminho de resolução (AUTORIDADE = a chain, sempre)

0. **Snapshot DNS autoritativo**: uma visão imutável
   (`Arc<HashMap<DomainId, {valid_until, has_record}>>`,
   feed `tokio::sync::watch`) publicada pelo núcleo do relay **após
   cada bloco aplicado** (push, não poll) e no boot.
   Decisões de existência/expiração — NXDOMAIN, domínio expirado — são
   servidas da RAM através deste snapshot: um clone de `Arc` + uma
   consulta ao map, **nunca entrando no run loop mono-proprietário do
   relay**. O caminho completo
   (`RelayCommand`/`ResolveLocal`) permanece reservado às buscas
   reais de registros (pulls DHT), raras graças ao cache. O
   snapshot NUNCA carrega dados de registro: apenas existência e
   expiração.
   - **Deltas copy-on-write O(tocados)**: o map é um
     `Arc<HashMap>`; uma extensão do tip publica o delta do bloco
     (as próprias transações do bloco nomeiam todo domínio que podem
     mudar — Register/Assign/Update/Renew/TransferDomain — mais as
     remoções do GC determinístico de expiração avaliadas no push)
     através de `Arc::make_mut`: o map é clonado SOMENTE se um
     leitor ainda detém a visão anterior. Boot e reorgs reconstruem o
     snapshot completo (um passe paginado pelo backend).
   - Frescor: um leitor pode ver um snapshot com algumas centenas de
     ms de idade durante a aplicação de um bloco — a mesma
     desatualização que um RPC de status teria; os TTLs do cache DNS
     não são afetados (as respostas permanecem autoritativas segundo
     sua origem).
1. **Cache fragmentado**: [`ShardedCache`] — 16 shards `Cache`
   independentes com chave `hash(qname)` (o qtype compartilha o shard:
   localidade por nome). Um nome quente disputa com 1/16 do
   tráfego em vez de enfileirar atrás de um único mutex a milhares de
   qps. Limites GLOBAIS (100.000 entradas / 64 MiB por padrão,
   `--dns-cache-limit` / `--dns-cache-max-mem`) divididos por shard
   (pior caso documentado: fator 1 + N/shards). O tamanho REAL do
   rdata de cada entrada é contabilizado (~128 bytes de overhead por
   entrada) e a evacuação FIFO dispara no primeiro dos dois
   limites atingidos — o teto de memória vale mesmo contra entradas
   de tamanho completo (`MAX_DHT_RECORD_LEN`). TTL 60 s
   (autoritativo) / limitado a 600 s (fallback). O cache negativo
   (NXDOMAIN/NODATA) segue as mesmas regras; cada entrada
   carrega sua origem (autoritativa ou fallback) para que o bit AA
   permaneça honesto num hit. **Nunca cachear SERVFAIL** (falha local
   transitória) **nem REFUSED** (estado de configuração): ambos são
   recomputados a cada consulta. Evacuação FIFO na capacidade.
   **Hot path (inspirado em Cloudflare/1.1.1.1)**:
   - entradas `Arc<CacheEntry>` compartilhadas: um hit é um bump de
     refcount, NUNCA um clone profundo do rdata (`put` SUBSTITUI o Arc —
     nenhuma mutação compartilhada; o TTL continua verificado sob a
     trava do shard);
   - **busca sem alocação**: map de dois níveis
     (`Box<str>` qname → qtype → entrada), o `&str` emprestado da
     consulta sonda diretamente (`Borrow<str>`) — a antiga
     chave de tupla `(String, u16)` alocava uma `String` nova a
     CADA consulta;
   - **registros contíguos empacotados**: um buffer `Box<[u8]>` por
     entrada (`u16 rdlen ‖ u16 rtype ‖ u32 ttl ‖ rdata` por
     registro) em vez de um `Vec` por registro — o orçamento de
     memória rastreia exatamente o que é alocado;
   - **compressão de nomes RFC 1035 na saída**: o owner das
     respostas autoritativas é um ponteiro de compressão de 2 bytes
     (`0xC00C`) para a pergunta ecoada — a pergunta nunca é
     recodificada integralmente por registro.
2. **Forma Scone?** Pré-filtro estrito: labels `[a-z0-9-_]`
   (1..63), ≥ 2 labels, TLD 1..63 bytes `[a-z0-9-]` (ver
   `TldName::MAX_LEN`), comprimento total ≤ 253 — **e TLD fora da
   raiz ICANN** (`is_icann_tld`, lista IANA congelada + RFC 2606:
   `com`, `fr`, `org`… jamais pertencem ao Scone).
   Um nome que NÃO PODE ser um nome Scone (por ex. `www.foo_bar`,
   underscore no TLD) vai ao fallback; um possível nome Scone
   (`absent.uip`) é assunto da chain: não registrado → NXDOMAIN
   autoritativo, mesmo sem upstream. Um TLD ICANN (`example.com`)
   é encaminhado aos upstreams se configurados, RECUSADO (REFUSED) caso
   contrário — nunca resolvido autoritativamente pelo Scone.
3. **Busca de apex**: sufixos do mais longo ao mais curto (mín.
   2 labels). O primeiro sufixo cujo estado on-chain é conhecido pelo
   relay = apex; o conjunto de registros serve o qname inteiro
   (subnomes incluídos — comportamento curinga implícito, decisão
   aberta).
4. **`resolve_local` (canal interno)**: um único round trip, nunca
   um waiter DHT. Estado on-chain (`registered`, owner, sequence) +
   registro do cache DHT local **re-verificado** (owner, sequence,
   record_hash == compromisso on-chain). Um registro não conforme
   nunca responde — disponibilidade ≠ autoridade.
   - não registrado → NXDOMAIN;
   - registrado sem registro válido → NODATA (NOERROR, 0
     respostas);
   - registro verificado → respostas filtradas por qtype.

## Codec (RFC 1035, subconjunto estrito)

- Consulta: header de 12 bytes, **uma** pergunta, QCLASS = IN, **sem
  compressão** (ponteiro = rejeitar). Labels em minúsculas e
  validados. QR=1, QDCOUNT≠1, labels inválidos → **silêncio**
  (lixo não custa nada).
- Resposta: QR|RD(echo)|RA, **AA apenas em respostas autoritativas**
  (chain, fresca ou em cache — nunca em fallback nem em RCODEs de
  erro), pergunta copiada verbatim, respostas autoritativas com
  **owner comprimido RFC 1035** (ponteiro de 2 bytes `0xC00C`
  apontando para a pergunta ecoada); as respostas do upstream de
  fallback são servidas tal qual (nunca recodificadas).
- Tipos servidos: A, AAAA, CNAME, NS, MX, TXT, **PTR, SRV, CAA**
  (+ Unknown recodificado verbatim; 9 tipos nativos no total, rdata
  codificado por RFC — 1035/2782/8659; CAA estrito: apenas bit
  crítico 0x80, tags em minúsculas). Uma consulta A/AAAA inclui os
  CNAMEs do conjunto (perseguição de alias). Consulta ANY (255): o
  conjunto inteiro filtrado pelo codec.
- Limites (M7b): **o orçamento de resposta depende do transporte** —
  512 bytes em UDP (payload clássico RFC 1035 §2.3.4, sem EDNS),
  65.535 em TCP. Um conjunto que excede o orçamento é cortado no
  último registro que couber, com **TC=1** (truncamento honesto, RFC
  1035 §4.2.1): o cliente repete em TCP (RFC 7766), onde o conjunto
  completo é servido. Uma resposta do upstream maior que o orçamento
  UDP é recodificada truncada (TC=1) em vez de entregue tal qual;
  em TCP passa byte a byte. O truncamento também se aplica a
  hits de cache (o orçamento é aplicado no momento da resposta, não
  no preenchimento do cache).
- RCODEs: 0 NOERROR (com ou sem respostas), 2 SERVFAIL (falha
  local), 3 NXDOMAIN (autoritativo), 5 REFUSED (não-Scone
  sem upstream).

## Fallback recursivo

- Upstreams `addr:port` (UDP), tentados em ordem, timeout de 2 s por
  upstream, socket **conectado** ao upstream (filtragem pelo
  kernel: um pacote de qualquer outra fonte nunca é aceito).
  A resposta do upstream só é aceita se **corresponder à
  consulta**: mesmo TXID, QR=1, pergunta (QNAME+QTYPE+QCLASS) copiada
  byte a byte. Uma resposta não conforme → SERVFAIL, e
  nada é cacheado.
- A resposta validada do upstream é repassada tal qual; suas
  respostas são cacheadas com origem `fallback` (AA nunca posto, mesmo
  num hit de cache) e TTL limitado a 600 s. RCODEs de erro do
  upstream (outros que NOERROR/NXDOMAIN) não são cacheados.
- **Privado por padrão**: sem upstream = REFUSED para nomes fora da
  forma Scone.

## Cache negativo DHT (anti-amplificação)

Um **pull DHT sem resposta** (registro esperado on-chain —
`record_hash` comprometido — mas ausente do cache local e dos
pares consultados) estaciona o nome por `DHT_NEGATIVE_TTL_SECS` (30 s)
num cache negativo limitado (FIFO, teto `DHT_NEGATIVE_CAP` = 4.096
nomes): resoluções repetidas do mesmo nome dentro do TTL são
**suprimidas** (contadas, nunca re-puxadas). Um scanner — ou um nome
quente cujo owner nunca publicou — não pode mais ampliar uma
consulta DNS num `GetRecord` por par da malha por consulta. A entrada
negativa morre no TTL, na chegada de um registro (o pull bem-sucedido
escreve no cache real), ou por ser empurrada para fora do teto FIFO.

Contadores expostos (`scone_dht_negative_hits_total`,
`_expired_total`, `_capped_total` — ver "Métricas" em `relay.md`).

## TCP (M7b, RFC 7766)

Mesma porta que o UDP (segundo bind TCP no endereço `--dns`). O
transporte segue a RFC 1035 §4.2.2: toda mensagem é prefixada por
seu comprimento em **2 bytes** big-endian.

- **Consulta**: prefixo de comprimento ≥ 1 e ≤ 512 bytes (uma
  consulta DNS cabe no payload clássico — RFC 7766 §6; nenhum EDNS
  servido). Fora dos limites → fechamento imediato da conexão.
- **Resposta**: orçamento de 65.535 bytes (o máximo do prefixo de 2
  bytes) — um conjunto grande demais para o UDP é servido **na
  íntegra** aqui, TC=0.
- **Pipelining**: múltiplas consultas por conexão são servidas em
  série, em ordem (uma escrita = uma resposta prefixada).
- **Timeout de leitura**: 5 s por leitura (prefixo ou corpo) — um
  cliente silencioso é desconectado, seu slot liberado.
- **Conexões concorrentes**: 64 no máximo (semáforo); a 65ª é
  rejeitada imediatamente (conexão fechada), nunca enfileirada —
  nenhum crescimento sem limites de tasks ou memória.
- Erros de leitura/escrita/timeout: log debug + fechamento limpo.
  **Nunca entrar em pânico sobre entrada de rede.**
- Lixo (consulta incompreensível) recebe silêncio mas não
  fecha a conexão: a próxima consulta (válida) é servida.

```text
client                     server
  │ [len u16][query]         │
  │ ─────────────────────►   │ cache → autoritativo → fallback
  │ ◄─────────────────────   │ [len u16][response ≤ 64 KiB]
  │ (o UDP tinha respondido TC=1 → o cliente repete aqui)
```

## Limites e guardrails

| Guardrail | Valor |
|---|---|
| Datagrama UDP (consulta/resposta) | 4096 bytes (rejeitado além) |
| Orçamento de resposta UDP (TC=1 além) | 512 bytes (sem EDNS) |
| Consulta TCP máxima | 512 bytes (fechada além) |
| Resposta TCP máxima | 65.535 bytes |
| Conexões TCP concorrentes | 64 (semáforo, rejeição imediata) |
| Timeout de leitura TCP | 5 s |
| Cache | limites GLOBAIS 100.000 entradas / 64 MiB por padrão, divididos em 16 shards (`--dns-cache-limit` / `--dns-cache-max-mem`) |
| TTL autoritativo | **por registro** (`record.ttl`, limites 1..=2.592.000 s; a constante histórica de 60 s permanece apenas para respostas de serviço) |
| TTL fallback (clamp) | 600 s |
| Consultas UDP paralelas | 4.096 (`MAX_INFLIGHT_UDP`, semáforo; além: drop, o cliente repete) — ajustável `--dns-max-inflight` (mín 1) |
| Cache negativo DHT | 30 s (`DHT_NEGATIVE_TTL_SECS`), teto FIFO 4.096 nomes |
| Timeout por upstream | 2 s |

Regras do projeto honradas: nunca entrar em pânico sobre dados de
rede; decodificação estrita em toda parte; o servidor não toca nem
chain, nem store, nem DHT (o relay permanece a única autoridade de
verificação).

## Testes

- Unitários (`scone-network::dns`): codec estrito (QR, compressão,
  rejeições de labels), respostas autoritativas, NODATA, NXDOMAIN,
  REFUSED sem upstream, fallback para um mock UDP, cache
  limitado, chunking TXT, parsing de upstream; fix-up M6: honestidade
  do bit AA (autoritativo vs fallback vs erros, incluindo hits de
  cache), truncamento TC=1 de um conjunto superdimensionado, rejeição
  de resposta do upstream (TXID ou pergunta divergente), nenhum cache
  de SERVFAIL/REFUSED; cache fragmentado: dispersão de shards, divisão
  dos limites globais, 4 workers concorrentes num nome quente; hot
  path: busca sem alocação, entradas empacotadas, compressão do owner.
- M7b: orçamento por transporte (UDP 512 bytes → TC=1, TCP → conjunto
  completo), truncamento de um hit de cache, `run_tcp` real (conjunto
  > 512 servido na íntegra, pipelining de 2 consultas, consulta
  superdimensionada → close, cliente silencioso → timeout de 5 s,
  lixo depois consulta válida na mesma conexão), resposta do
  upstream > 512 recodificada TC=1
  em UDP / byte a byte em TCP.
- Integração real (`tests/dns_server.rs`): um relay completo
  (incluindo produção devnet) + registro + UpdateDomain +
  PutRecord, depois consultas UDP reais: A verificado (bit AA), TXT,
  NODATA AAAA, NXDOMAIN, fallback via mock, nova consulta, subnome via
  apex.
- E2E binário (`crates/scone/tests/dns_cli.rs`): o binário `scone`
  real — `relay --dns`, `identity generate`, `domain register`,
  `domain update --file`, depois `dig` (dnsutils) (A verificado, TXT,
  NXDOMAIN, REFUSED) numa porta UDP real.

## Decisões abertas

- Nenhum EDNS (o orçamento UDP fica em 512 bytes; o truncamento TC=1 +
  TCP cobre a necessidade da devnet).
- O conjunto de registros não carrega nome por registro: o conjunto
  inteiro serve todos os subnomes do apex (curinga implícito). Um
  endereçamento por registro exigiria estender o `DnsRecord`
  (protocolo).
- O fallback encaminha a consulta bruta: um upstream malicioso pode
  responder qualquer coisa (resposta não validada — este é o modelo
  recursivo padrão; os nomes Scone NUNCA passam por ele).

## GeoDNS (roteamento geográfico)

> Especificação normativa. Qualquer divergência doc/código = bug.

### Tabela GeoIP embutida (automática)

Todo binário embute uma **tabela de mapeamento IP → país** compilada
no relay: **nenhuma configuração necessária**, o roteamento geográfico
está ativo em todo lugar, automaticamente.

- **Fonte de dados**: **DB-IP Lite** ("IP to Country Lite"),
  licenciado sob **CC BY 4.0** — redistribuível com atribuição:
  `https://db-ip.com` (curado mensalmente; antes: as delegações
  dos RIR via `scripts/gen-geo-table.py`, removido). Células
  quantizadas **/20 v4 e /24 v6** (a granularidade nativa do
  DB-IP, ~1.3 M de faixas, excede o limite de memória do parser)
  — ~96.500 regras. A data de geração está no header da tabela
  (`assets/embedded-geo.table`).
- **Checksum**: o SHA-256 da tabela é calculado em tempo de
  compilação (`build.rs`), **reverificado na partida**
  (mismatch = recusa tipada — um binário alterado nunca serve
  respostas geográficas) e logado na partida com a atribuição. O
  hash garante que "todos os nós de um mesmo build compartilham a
  mesma tabela" é VERIFICÁVEL, não presumido.
- **Regeneração**: `scripts/geo-convert.py` (baixa o CSV de
  DB-IP Lite do mês, converte — país numérico, continente derivado
  ISO→M49, ASN/região ausentes da Lite).
- **Granularidade**: país (ISO 3166-1 numérico) + continente (ONU
  M49) derivado. Sem ASN nem região (ausentes da Lite — uma
  necessidade de ASN/região aguarda uma evolução do pipeline
  embutido, nunca uma tabela local).
- **Desempenho**: consulta por **busca binária** sobre faixas
  ordenadas (índice construído uma vez no primeiro uso, por família
  v4/v6) — medido ~95 ns/consulta a quente, ~50 µs em acesso
  aleatório na tabela completa
  (bench: `cargo test -p scone-network --release --test geo_bench
  -- --ignored`).
- **Autolocalização do relay**: o roteamento segue o **IP de origem
  de cada consulta** — um relay de LAN encaminha seus clientes
  locais à zona certa automaticamente.
- **Tabela IMPOSTA** (remove-geo-operator-table, 2026-10-22): a
  tabela embutida é a **única fonte** — não existe import nem
  sobreposição do operador. **Não existe opção geo em runtime**:
  toda flag `--geo-*` é uma recusa clap seca (a geozone é imposta
  pelo build). Um operador com necessidades específicas (ASN,
  região) aguarda uma evolução do pipeline embutido, não uma
  tabela local.

### Visão geral

Um domínio pode publicar regras de **roteamento geográfico** no seu
conjunto de registros assinado: registros `GEOROUTE` (tipo privado
Scone `32.768`, fora da faixa IANA). O resolvedor responde perguntas
A/AAAA para esses domínios com **o endereço do endpoint escolhido pela
regra** (escopo mais próximo da posição do CLIENTE, derivado do seu
IP de origem via tabela), em vez de um registro A/AAAA estático.
Todo o resto (domínios sem GEOROUTE, outros qtypes) segue o
caminho DNS clássico, **byte a byte**.

### Métricas GeoDNS

O endpoint Prometheus expõe a divisão das respostas geo:

- `scone_dns_geo_answers_total` — total de respostas GEOROUTE servidas;
- `scone_dns_geo_by_continent_total{continent="<M49>"}` — por continente;
- `scone_dns_geo_by_country_total{country="<ISO>"}` — por país
  (limitado pelos próprios padrões: ≤ 9 continentes, ≤ 250
  países).

### Registro GEOROUTE (wire)

Um GEOROUTE = um **escopo** + ≤ 32 **endpoints** (ordenção canônica na
decodificação: prioridade crescente, peso decrescente, alvo crescente;
duplicata = rejeitar):

```text
GeoScope   = 0x00 (world)
           | 0x01 continent.v (UN M49 code: AF=2, AS=142, EU=150,
                                NA=3, SA=5, OC=9, AN=999)
           | 0x02 country.v   (ISO 3166-1 numeric code: US=840,
                                TW=158, FR=250…)
           | 0x03 region.v    (OPAQUE ISO 3166-2 subdivision, u32:
                                publisher-defined number, compared
                                as is — no local table)
           | 0x04 asn.v       (AS number, u32)
           | 0x05 prefix.addr[16] || prefix.len u8
             (normalized 16-byte IP prefix: IPv4 = address in the
              first 4 bytes + ZERO tail, len ≤ 32;
              IPv6 = 16 free bytes, len ≤ 128; host bits ALWAYS
              zero — a non-normalized prefix is rejected)
GeoTarget  = 0x00 octets[4]  (binary IPv4 address)
           | 0x01 octets[16] (binary IPv6 address)
GeoEndpoint = target || priority.v || weight.v
GEOROUTE    = scope || count.v(≤32) || endpoints…
```

Limites de validação (decodificação): `priority > 0`, `weight ≤ 100`
(`weight = 0` = endpoint inativo, nunca eleito), TTL dentro de
1..=2.592.000 s como qualquer registro. Um país/continente desconhecido
das tabelas locais é **permitido** no wire: o roteamento simplesmente
retorna a um escopo superior.

**Compatibilidade retroativa do wire (v1 → v2)**: os bytes v1 estão
INALTERADOS — os escopos v1 (mundo/continente/país) mantêm seus
discriminantes `0x00..=0x02` e sua codificação exata; os três
novos escopos tomam os discriminantes seguintes (`0x03..=0x05`) numa
ordem total consistente com a ordenação canônica
(`World < Continent < Country < Region < Asn < IpPrefix`). Um
decodificador v1 encontrando um discriminante ≥ `0x03` já falhava
estritamente (registro rejeitado); os bytes produzidos em v1 continuam
decodificados identicamente (ver o teste `geo_known_bytes`,
congelado).

### Seleção (resolução)

1. **Escopo** — especificidade decrescente, **6 níveis**:
   **prefixo IP mais longo correspondente → AS exato → país exato →
   região exata → continente → mundo**.
   - *IpPrefix*: entre as rotas cujo prefixo corresponde ao
     IP do cliente (tal como conhecido pelo localizador: `client_ip`,
     caso contrário os prefixos de origem conhecidos), o **mais
     longo** (`len` máximo) vence; em `len` igual, o primeiro na
     ordem canônica (a ordem do slice `routes`, ordenado na
     codificação). Uma
     subdivisão é menos específica que um prefixo/AS (que
     identificam a rede exata) e menos que o país
     (numeração opaca, confiável apenas se o publicador e o
     localizador partilharem a mesma), mas mais que o continente
     (ela está contida no seu país, logo a fortiori no seu
     continente).
   - *Asn*: número AS exato do cliente.
   - *Country*: país exato.
   - *Region*: subdivisão exata (identificador opaco).
   - *Continent*: fornecido pelo localizador **ou derivado do
     país** via tabela local (`continent_of_country`); um
     país desconhecido da tabela retorna ao continente explícito
     ou ao mundo.
   - *World*.
   Um [`GeoLocation`] v1 (região/asn/prefixos/client_ip não definidos)
   produz exatamente o comportamento v1: país → continente →
   mundo. Nenhuma rota correspondente (inclusive sem escopo World) →
   comportamento DNS clássico, como se o GEOROUTE não existisse.
2. **Prioridade** — semântica tipo SRV: apenas o grupo de **prioridade
   mínima** é elegível; prioridades maiores são níveis de
   **failover**.
3. **Peso** — dentro do grupo eleito, sorteio **ponderado
   determinístico**: hash estável (SplitMix64) de um nonce (o
   contador de consultas do resolvedor) contra os pesos cumulativos. Sem
   RNG: duas consultas com o mesmo nonce dão o mesmo resultado,
   nonces crescentes distribuem proporcionalmente aos pesos.
   `weight = 0` = inativo, nunca eleito (mesmo sozinho — passar ao
   nível seguinte).
4. **Ordem assinada, zero sondagem** — o resolvedor NUNCA julga a
   disponibilidade de um endpoint: todo endpoint ativo (weight > 0)
   do escopo eleito é candidato, a cabeça do nível mínimo é eleita
   pelo sorteio ponderado, e os seguintes atuam como failovers
   **na ordem do conjunto de registros assinado** (a ordem É o
   mecanismo de failover). A alcançabilidade de um endpoint é
   assunto do operador desse endpoint, nunca do resolvedor.

### Resposta

- Um único endereço: **A se IPv4, AAAA se IPv6** — filtragem
  estrita por família segundo o qtype (uma pergunta A NUNCA
  responde um IPv6, e vice-versa).
- TTL = o do registro GEOROUTE de origem (limitado na decodificação
  pelas regras gerais de TTL).
- AA posto (resposta autoritativa — o GEOROUTE é um registro assinado
  verificado como qualquer outro).
- Resposta vazia após filtragem (nenhum candidato da família certa,
  nenhuma rota para a zona) → o resolvedor
  continua no seu caminho clássico (tipicamente NODATA: NOERROR, 0
  respostas); rcodes de erro seguem as regras gerais (nunca cacheados).
- Um registro A/AAAA clássico coexistindo no conjunto leva
  PRECEDÊNCIA sobre o geo (o geo aplica-se apenas na ausência de uma
  resposta wire para o qtype).

### Cache: decisão explícita

A decisão de roteamento geo **NUNCA é cacheada** sob
`(qname, qtype)`: ela depende do escopo do cliente do resolvedor, e
tal chave serviria a rota US a um cliente TW. As respostas geo são
recomputadas a cada consulta (custo: um select_scope + sorteio
ponderado, µs) e contornam o cache. O cache clássico (qname, qtype)
permanece inalterado para todo o resto — incluindo o NODATA
produzido quando o geo não decide nada. Os rcodes de erro permanecem
nunca cacheados.

### Delegação NS (registros `ns`) — o owner escolhe quem serve

O owner de um domínio escolhe o modo de serviço por domínio:

- **DHT (padrão)** — sem registro NS, a rede Scone serve os
  A/AAAA on-chain (relay `--dns`, ou a ponte NSD via export);
- **NS independente** — o owner adiciona registros
  `ns <ip-ou-nome>` (SEUS nameservers, IP direta ou hostname): o
  domínio fica DELEGADO — a rede deixa de servir seu conteúdo
  (A/AAAA → NODATA + autoridade NS), e quem resolve segue os NS do
  owner. O limite: **12 NS distintos por domínio** (todas as
  formas, deduplicados por valor — espelho dos caps dos registros
  reais). Remover um NS libera o slot; remover todos restaura o
  modo DHT.

**Não existe verificação de saúde**: a disponibilidade dos
endpoints é assunto dos operadores dos endpoints — o resolvedor
serve o conjunto assinado como está, na ordem dos registros (a
ordem dos registros é o mecanismo de failover).

### Segurança

- Registros GEOROUTE são registros assinados como qualquer outro:
  verificados contra a chain (owner, sequence, record_hash) antes de
  qualquer uso. **Nenhuma confiança de rede sem validação** — um
  GEOROUTE DHT não verificado nunca responde.
- A posição do cliente vem do **circuito autodescoberto**: o
  relay se autolocaliza via seu endereço externo (STUN/AutoNAT/
  UPnP) resolvido pela tabela embutida imposta pelo build —
  clientes LAN (IP de origem privada) herdam essa posição,
  clientes da Internet são resolvidos pelo próprio IP, o
  desconhecido cai no World. Nunca de uma API GeoIP de rede
  (nenhum vazamento de consulta, nenhuma dependência) e nunca de
  uma declaração do operador (a geozone é imposta pelo build).
- O sorteio é determinístico e local: nenhum segredo, nenhuma
  imprevisibilidade exigida (a segurança é a da assinatura do
  registro, não a do sorteio).

### Exemplo completo (spec de referência)

`www.example.scone` publica (apex `example.scone`, o conjunto
inteiro serve todos os subnomes):

| Escopo | Endpoint | Prioridade | Peso |
|---|---|---|---|
| US (840) | 10.1.0.1 | 10 | 70 |
| US (840) | 10.1.0.2 | 10 | 30 |
| US (840) | 10.2.0.1 | 20 | 100 |
| TW (158) | 10.3.0.1 | 10 | 80 |
| TW (158) | 10.4.0.1 | 10 | 20 |
| TW (158) | 10.2.0.1 | 20 | 100 |
| WORLD | 10.0.0.1 | 100 | 100 |

Comportamento esperado:

- cliente **US** → 10.1.0.1 (≈70%) ou 10.1.0.2 (≈30%); o nível p20
  só é alcançado se nenhum endpoint p10 estiver ativo (weight = 0
  em todos) → então **10.2.0.1**;
- cliente **TW** → 10.3.0.1 (≈80%) ou 10.4.0.1 (≈20%);
- cliente **FR** (250) → nenhuma rota FR ou EU no conjunto → WORLD →
  **10.0.0.1**;
- pergunta **A** numa rota somente IPv6 → resposta vazia (nunca um
  IPv6 em A);
- domínio sem regra aplicável → mesma pergunta, comportamento clássico (NODATA:
  o GEOROUTE não tem rdata A no wire).

### Opções CLI (`scone relay`)

| Opção | Padrão | Papel |
|---|---|---|
| (nenhuma — GeoDNS sempre ativo, tabela embutida) | on | Roteamento automático por IP de origem |
| `--geo-ip-table`, `--geo-config`, `--geo-country/…`, `--health-*` | — | (REMOVIDAS — quebra seca: a geozone é imposta pelo build e o resolvedor não faz nenhuma verificação de saúde; não existe nenhuma flag em runtime) |

#### A tabela embutida é a única fonte

A tabela GeoIP é **imposta pelo build** (DB-IP Lite, quantizada,
veja a seção da tabela embutida acima): nenhuma tabela de operador
pode ser importada. Resolução: o **IP de origem da consulta DNS**
(por cliente: UDP `recv_from` / TCP `peer_addr`) é comparado em 16
bytes contra a tabela embutida; a regra com o **prefixo
correspondente mais longo** vence (empate: a primeira na ordem do
arquivo).

- **origem privada/irroteável** (RFC1918, ULA fc00::/7,
  link-local, loopback): o cliente está DENTRO da rede do relay —
  ele herda a posição autodescoberta do próprio relay;
- **origem pública ausente da tabela**: World (sem divisão geo —
  nunca uma resposta errada).

**O rigor** é preservado para qualquer futura superfície de
tabela: uma linha inválida é um erro de partida tipado carregando
o número da linha ofensiva; nunca uma linha silenciosamente
ignorada, nunca uma tabela parcial.

#### Precedências das opções

- **Localizador**: a tabela embutida IMPÕE a geolocalização —
  nenhuma sobreposição do operador. O relay descobre sua posição
  a partir do seu endereço externo (STUN/AutoNAT/UPnP resolvidos
  pela tabela); clientes LAN herdam essa posição; clientes da
  Internet são resolvidos pelo seu IP de origem; o desconhecido
  cai no World.

Exemplo (o roteamento geo está ativo sem NENHUMA flag geo):

```bash
scone relay --dns 127.0.0.1:5353
```


## DoH / DoT (transportes criptografados) — via frontend externo

O servidor DNS do Scone fala **UDP e TCP puros** (RFC 1035 / 7766)
— este é o escopo nativo, completo e testado. Os transportes
criptografados até o *cliente final* (DoH — DNS-over-HTTPS, RFC 8484;
DoT — DNS-over-TLS, RFC 7858) são **delegados a um frontend
padrão**: o Scone serve os dados, o frontend serve o
transporte.

Por que este design: um servidor DoH nativo exigiria uma pilha HTTPS
completa dentro do nó (certificados, renovação, HTTP/2) — uma grande
superfície operacional por zero benefício antes que haja usuários
finais no lado dos navegadores. Frontends existentes o fazem melhor e
há mais tempo; o nó permanece mínimo.

### Exemplo: unbound fazendo DoH na frente de um relay Scone

`/etc/unbound/unbound.conf`:

```
server:
    interface: 127.0.0.1
    port: 5354
    do-ip4: yes
    do-udp: no
    do-tcp: yes

    # DoH (RFC 8484) on 443
    https-port: 443
    tls-service-key: "/etc/unbound/scone.key"
    tls-service-pem: "/etc/unbound/scone.pem"

forward-zone:
    name: "."
    forward-addr: 127.0.0.1@5353   # the Scone relay (--dns 127.0.0.1:5353)
```

O certificado (`scone.pem`/`scone.key`) é obtido como qualquer serviço
TLS — por ex. certbot/acme.sh no domínio do resolvedor. Os clientes
DoH então apontam para `https://dns.example.tld/dns-query`.

### Exemplo: knot-resolver fazendo DoT

`/etc/knot-resolver/kresd.conf`:

```
net.listen('127.0.0.1', 5353, { kind = 'dns' })
net.listen('0.0.0.0', 853, { kind = 'tls' })
tls.cert_file = '/etc/knot-resolver/scone.pem'
tls.key_file  = '/etc/knot-resolver/scone.key'
```

### O que o frontend acrescenta

- **Criptografia do canal cliente→frontend** (o segmento hostil:
  a rede do usuário); o segmento frontend→relay permanece
  local (mesma máquina ou rede confiável).
- **Um cache extra no lado do frontend** (cache do unbound/knot
  também — inofensivo, os TTLs do Scone permanecem o teto).
- **Um endpoint estável** (IPv4/IPv6, domínio) enquanto os relays se
  movem.

### O que o frontend NÃO faz

- Ele não *verifica* nada: toda resposta servida pelo relay já está
  verificada contra a chain antes de ser cacheada (ver
  "Caminho de resolução"). O frontend repassa respostas já
  autoritativas.
- Ele não substitui a descentralização: um frontend DoH é um serviço
  *operado* — a escolha do operador do resolvedor, não a do
  protocolo. Cada relay Scone permanece utilizável de forma
  independente sobre UDP/TCP nativos.

### Implementação nativa futura (documentada, não comprometida)

Um transporte DoH nativo (`--doh ADDR`) permanece concebível: o
formato wire sendo idêntico (a RFC 8484 reutiliza mensagens DNS
binárias), seria um decode POST/GET + HTTP/2 atrás de um proxy
TLS (caddy/nginx tratando os certificados), o nó em si permanecendo
sem TLS. Nenhum trabalho em curso — a demanda virá com os usuários
finais.
