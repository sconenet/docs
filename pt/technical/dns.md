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

- **Fonte de dados**: os **arquivos de delegação dos RIR**
  (RIPE NCC, ARIN,
  APNIC, LACNIC, AFRINIC), livremente baixáveis e redistribuíveis:
  `https://ftp.ripe.net/pub/stats/<rir>/delegated-<rir>-extended-latest`
  (país de registro de cada bloco IPv4/IPv6 delegado). A data de
  geração está no header da tabela
  (`assets/embedded-geo.table`).
- **Regeneração**: `scripts/gen-geo-table.py` (baixa os 5
  arquivos, converte, normaliza prefixos — ~176.000 regras).
- **Granularidade**: país (ISO 3166-1 numérico) + continente (ONU
  M49) derivado. Sem ASN nem região (ausentes das delegações).
- **Desempenho**: consulta por **busca binária** sobre faixas
  ordenadas (índice construído uma vez no primeiro uso, por família
  v4/v6) — medido ~95 ns/consulta a quente, ~50 µs em acesso
  aleatório na tabela completa
  (bench: `cargo test -p scone-network --release --test geo_bench
  -- --ignored`).
- **Autolocalização do relay**: o roteamento segue o **IP de origem
  de cada consulta** — um relay de LAN encaminha seus clientes
  locais à zona certa automaticamente.
- **Precedência**: uma tabela explícita de operador
  (`--geo-ip-table`/geo-config) **sobrepõe** a embutida.

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
4. **Failover de saúde** — um endpoint **Unhealthy** é posto de lado;
   se todo o nível mínimo estiver indisponível, passar ao nível de
   prioridade seguinte. **Desconhecido conta como disponível** (o
   desconhecido não é penalizado). Tudo indisponível →
   comportamento DNS clássico.

### Resposta

- Um único endereço: **A se IPv4, AAAA se IPv6** — filtragem
  estrita por família segundo o qtype (uma pergunta A NUNCA
  responde um IPv6, e vice-versa).
- TTL = o do registro GEOROUTE de origem (limitado na decodificação
  pelas regras gerais de TTL).
- AA posto (resposta autoritativa — o GEOROUTE é um registro assinado
  verificado como qualquer outro).
- Resposta vazia após filtragem (nenhum candidato da família certa,
  tudo Unhealthy, nenhuma rota correspondente) → o resolvedor
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

### Health checks (`--health-check`)

- `HealthCache` por alvo (`GeoTarget`): `probe` = leitura não
  bloqueante (status fresco, ou Unknown), **nunca** uma verificação
  disparada do caminho de resolução.
- Refresh: se um candidato eleito está Unknown/ausente do
  cache, UM refresh é agendado como task de fundo
  (`tokio::spawn_blocking`: o connect TCP é bloqueante,
  proibido no loop UDP). Anti-spam: um refresh por vez por
  alvo (HashSet), reiniciável após o refresh anterior
  terminar.
- `TcpChecker`: connect TCP limitado (timeout de 2 s) a
  `target:port`. **A porta não pode ser derivada de um registro
  DNS**: `--health-port` a define (padrão 443). O checker é
  **deliberadamente ingênuo** (um connect bem-sucedido ≠ serviço
  saudável) — melhorável (HTTP, TLS, nível de aplicação) sem
  mudança de formato.
- TTL da entrada: `--health-ttl-secs` (padrão 5 s); entrada expirada =
  Unknown = disponível.

### Segurança

- Registros GEOROUTE são registros assinados como qualquer outro:
  verificados contra a chain (owner, sequence, record_hash) antes de
  qualquer uso. **Nenhuma confiança de rede sem validação** — um
  GEOROUTE DHT não verificado nunca responde.
- A posição do cliente vem da **tabela GeoIP embutida**
  (delegações RIR, ver acima) derivada do seu IP de origem — ou
  da configuração local explícita do resolvedor
  (`--geo-ip-table` ou
  `--geo-country`/`--geo-continent`/`--geo-asn`/`--geo-region`,
  que tem precedência), nunca de uma API GeoIP de rede (nenhum
  vazamento de consulta, nenhuma dependência).
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

- cliente **US** → 10.1.0.1 (≈70%) ou 10.1.0.2 (≈30%); se ambos
  estiverem Unhealthy → failover p20 → **10.2.0.1**;
- cliente **TW** → 10.3.0.1 (≈80%) ou 10.4.0.1 (≈20%);
- cliente **FR** (250) → nenhuma rota FR ou EU no conjunto → WORLD →
  **10.0.0.1**;
- tudo Unhealthy → nenhuma decisão geo → NODATA (NOERROR, 0
  respostas);
- pergunta **A** numa rota somente IPv6 → resposta vazia (nunca um
  IPv6 em A);
- domínio sem regra aplicável → mesma pergunta, comportamento clássico (NODATA:
  o GEOROUTE não tem rdata A no wire).

### Opções CLI (`scone relay`)

| Opção | Padrão | Papel |
|---|---|---|
| (nenhuma — GeoDNS sempre ativo, tabela embutida) | on | Roteamento automático por IP de origem |
| `--geo-ip-table <path>` | — | **Tabela GeoIP local** (ver abaixo). Tem PRECEDÊNCIA sobre todas as opções estáticas `--geo-*` |
| `--geo-country <u16>` | — | País do cliente, ISO 3166-1 numérico (840=US…) — estático |
| `--geo-continent <u16>` | — | Continente do cliente, ONU M49 (150=EU…); derivado do país se omitido — estático |
| `--geo-asn <u32>` | — | AS do cliente (64500…) — estático, novo na v2 |
| `--geo-region <u32>` | — | Subdivisão do cliente (identificador opaco do publicador) — estático, novo na v2 |
| `--health-check` | off | Ativa as verificações (failover de saúde) |
| `--no-health-check` | off | **Desativa explicitamente** as verificações — VENCE sobre `--health-check` (ver precedências) |
| `--health-ttl-secs <u64>` | 5 | TTL de uma entrada de saúde |
| `--health-port <u16>` | 443 | Porta TCP das verificações (não derivável de um registro DNS) |

#### Tabela GeoIP local (`--geo-ip-table`)

Arquivo TEXTO local de prefixos, lido **uma vez** na partida (100.000
regras no máximo, `MAX_GEOIP_RULES`). Uma regra por linha, `#` =
comentário, linhas vazias ignoradas:

```text
# prefix/mask        country continent asn     region
10.0.0.0/8           840   3          .       .
2001:db8::/32        250   150        64500   11
192.0.2.0/24         158   9
```

- `prefix/mask`: IPv4 (`len ≤ 32`, convertido à forma de 16 bytes,
  cauda zero — mesma convenção que o wire) ou IPv6 (`len ≤ 128`).
  **Os bits de host devem ser zero** (`10.1.2.3/8` é recusado);
- `country`: ISO 3166-1 **numérico** `u16`, obrigatório;
- `continent`: ONU M49 `u16`, opcional (`.` ou omitido = ausente,
  o continente pode então ser derivado do país);
- `asn`: `u32`, opcional (`.` = ausente);
- `region`: `u32` opaco, opcional (`.` = ausente).

Resolução: o **IP de origem da consulta DNS** (por cliente:
UDP `recv_from` / TCP `peer_addr`; desconhecido → IP de referência do
resolvedor) é comparado em 16 bytes; a regra com o **prefixo
correspondente mais longo** vence (empate: a primeira na ordem do
arquivo); nenhuma correspondência → posição vazia (World).

**Rigor**: o arquivo é configurado pelo operador — uma linha
inválida (prefixo malformado, país não numérico, continente
desconhecido, campo em excesso…) é um **erro de partida** carregando o
**número da linha ofensiva**; nunca uma linha silenciosamente ignorada,
nunca uma tabela parcial.

#### Precedências das opções

- **Localizador**: `--geo-ip-table` **tem precedência** sobre as
  opções estáticas
  (`--geo-country`/`--geo-continent`/`--geo-asn`/`--geo-region`),
  que são ignoradas quando a tabela é fornecida. Sem tabela,
  aplicam-se as opções estáticas.
- **Saúde**: `--no-health-check` é uma **flag dedicada que
  vence** sobre `--health-check` — a combinação
  `--health-check --no-health-check` deixa as verificações
  desativadas, sem ambiguidade de ordem de entrada. Sem
  `--health-check`, as verificações estão
  desativadas por padrão: **Unknown = disponível**, a resposta geo
  sai mesmo que o alvo esteja morto. (Um `NoopChecker` — sempre
  Healthy — também existe para montar a estrutura de saúde sem
  pôr de lado nenhum endpoint: testes/paranoia.)

Exemplo:

```bash
scone relay --dns 127.0.0.1:5353 \
            --geo-ip-table /etc/scone/geoip.txt \
            --no-health-check
```

Exemplo estático v2:

```bash
scone relay --dns 127.0.0.1:5353 \
            --geo-country 250 --geo-asn 64500 --geo-region 11 \
            --health-check --health-port 443
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
