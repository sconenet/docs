# CLI (`scone`) — log e verbosidade

> Comportamento normativo do log do binário `scone`. Qualquer divergência
> doc/código = bug (corrigir a fonte da verdade).

## Princípios

- **stdout é reservado à saída de comandos** (`scone status`,
  `identity list`, `status`, …). Um script pode redirecionar stdout
  sem nunca encontrar logs nele — o piping permanece confiável. Exceção
  aceita: `scone relay` emite as linhas de dados
  `p2p: <multiaddr>`, `dns: <addr>` e `metrics: <addr>` no
  stdout assim que os listeners estão vinculados
  (dados utilizáveis para `--bootstrap` / configuração, não logs;
  emitidos exatamente uma vez cada, com flush imediato).
- **Todos os logs vão para stderr** (formato `tracing` compacto),
  incluindo os do relay — o relay já logava no stderr antes da
  migração.
- As mensagens de erro voltadas ao usuário da CLI (`scone: <erro>`)
  ficam no stderr: isto é UX, não logs.

## Níveis e `--verbose` / `-v`

| Invocação | Nível efetivo |
|---|---|
| `scone <cmd>` (sem `-v`) | `warn` |
| `scone relay` (sem `-v`) | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …` (e mais) | `trace` |

`-v`/`--verbose` é uma flag **global e repetível**: é
aceita antes ou depois do subcomando (`scone -vv identity list`
e `scone identity list -vv` são equivalentes).

**Por que `scone relay` permanece em INFO por padrão**: o relay é um
daemon — ele deve logar sua atividade (escuta RPC/p2p, blocos aceitos e
produzidos, pares conectados, rejeições) para ser operável, sem exigir
uma flag. Os comandos one-shot, por outro lado, permanecem silenciosos
(WARN) para não poluir um terminal ou um script.

## `RUST_LOG` (override)

Se a variável de ambiente `RUST_LOG` estiver definida, ela **substitui
integralmente** o nível derivado de `-v` (sintaxe `env-filter`):

```bash
RUST_LOG=debug scone identity list        # debug sem -v
RUST_LOG=error scone -vv relay            # error vence sobre -vv
RUST_LOG=scone_network=trace,info scone relay   # por alvo
```

## Configuração de ambiente (`.env`)

Antes de analisar os argumentos, `scone` carrega um arquivo `.env` do
diretório atual (`dotenvy::dotenv()` em `main.rs`). Arquivo ausente
= no-op silencioso, nunca um erro. Os valores carregados tornam-se a
linha de base ambiental das opções do relay que declaram uma variável
de ambiente (`env = "…"` em `cli.rs`).

Precedência (uma opção, três fontes):

```text
explicit CLI flag  >  .env variable  >  built-in default
```

- uma flag passada na linha de comando SEMPRE vence;
- caso contrário a variável (seja carregada do `.env` ou definida no
  ambiente real do processo — mesma variável, mesmo efeito);
- caso contrário o padrão embutido do clap.

Variáveis repetíveis (`SCONE_BOOTSTRAP`, `SCONE_DNS_UPSTREAM`)
acumulam: uma linha por valor no `.env` (dotenvy anexa).

A tabela completa das variáveis `SCONE_*` (rede, escuta P2P,
bootstrap, gênese, UPnP, superfície DNS, métricas, perfil de
topologia, âncora) e a semântica de cada uma vive em
[`relay.md`](relay.md#environment-configuration-env); o
modelo comentado é o `.env.example` na raiz do repositório (copiar
para `.env`, nunca comitar — `.env` é gitignored).

## Semântica dos níveis (convenção de código)

- `error!`: falhas (nenhum call site hoje — falhas de rede são
  engolidas como `warn!` e apenas falhas locais fatais surgem como um
  erro tipado de `run()`);
- `warn!`: entradas de rede hostis ou maliciosas rejeitadas,
  txs velhas evacuadas do mempool, txs conflitantes descartadas
  durante a produção, falhas de saída/entrada, divergência de chave DHT;
- `info!`: bloco aceito/produzido, escuta RPC/p2p, par
  conectado, partida do relay (checkpoints de estado);
- `debug!`: detalhes internos (tx aceita, par desconectado, ida e volta
  RPC, listagem do keystore, eventos kad sob
  `SCONE_KAD_DEBUG`);
- `trace!`: reservado a payloads (nenhum call site hoje).

As crates de biblioteca (`scone-network`, …) emitem macros `tracing` mas
**inicializam nenhum subscriber**: a inicialização vive apenas no
binário `scone` (antes do despacho de subcomandos — o relay precisa de
seus logs desde o boot).


## Uso simples (cli-simplify-v1)

- **Sem `--peer` necessário**: os comandos falam com o relay local por
  padrão; env `SCONE_PEER` ou `--peer` sobrepõe.
- **Sem `--identity` necessário** quando o keystore detém uma única
  identidade (usada silenciosamente; várias → o erro as lista).
- Removidos (os stubs sugerem a substituição): `show`, `submit`,
  `lookup`, `owned`.
- Novos: `scone domain renew <name>`, `scone domains`, `scone domain
  get|set` (registro único, sem arquivo).

```bash
scone identity generate me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone domain renew example.scone
```

## Ciclo de vida dos TLDs (cli-unified-lifecycle)

`scone tld register <tld>` é o ponto de entrada de TLD em um comando,
simétrico ao `scone domain register` — o estado on-chain é
lido PRIMEIRO, depois a ação certa executa:

- **TLD livre** → registrado, nasce **ABERTO por padrão** (sem flag:
  namespace em autoatendimento, 1/3 do PoW). `--private` opta por
  um namespace possuído ao preço integral do PoW, com confirmação
  explícita (decisão econômica). `--yes` pula confirmações
  (scripts).
- **seu, expirando em 90 dias** → a RENOVAÇÃO é proposta
  (PoW vinculado à expiração atual — anti-reuso) e executa após
  confirmação.
- **seu, longe da expiração** → nada a fazer (`tld renew --force`
  para renovar mesmo assim).
- **de outro** → erro tipado limpo com o owner.

```bash
scone tld register mytld            # nascimento aberto (padrão)
scone tld register mytld --private  # namespace possuído (confirmar)
scone tld renew mytld               # renovação explícita
```

Da mesma forma, `scone domain register <name>` num nome que você possui e
que expira em 90 dias propõe a renovação em vez de falhar.

## Delegados de records (KSK/ZSK-lite)

O owner de um domínio pode conceder a OUTRAS chaves o direito de
assinar `domain update` em seu nome — o conjunto de delegados de
records (no máximo 3 chaves, substituição total a cada `set`):

```bash
# concessão (assinada pelo owner; o CLI busca a geração CAS do
# namespace e liga a transação a ela):
scone domain delegates set example.uip \
  --keys <pk-hex-1>,<pk-hex-2> --identity alice

# consulta (sem identidade):
scone domain delegates list example.uip

# revogar tudo:
scone domain delegates clear example.uip --identity alice
```

Um delegado então assina `domain update` exatamente como o owner
faria (`--identity deleg`) — on-chain o campo `owner` da tx é o
owner do DOMÍNIO enquanto a assinatura vem do delegado. A concessão
é exclusiva do owner (um delegado não pode gerenciar o conjunto),
esvaziada por uma transferência, e cada `set`/`clear` incrementa o
contador de mutações do namespace (CAS anti-replay — ver
`transactions.md`): um `set` rejugado não pode ressuscitar chaves
revogadas. A update de um delegado revogado é recusada localmente
(«neither the owner nor a record delegate»).

## Comandos avançados

Os padrões acima cobrem o uso cotidiano. Para scripts e interoperabilidade
web/API a superfície completa permanece disponível:

cada ação tem exatamente UM caminho canônico; os antigos aliases
de raiz (`show`, `lookup`, `owned`, `renew`) foram removidos
(remove-cli-aliases), assim como `domain info`, que nunca
correspondeu a um comando real.

A superfície hex offline (`tx build|sign|verify`, subcomandos
`submit tx`) foi REMOVIDA (remove-tx-surface-v2): as operações M8b
que ela servia são agora comandos de primeira classe —
`scone tld open <tld>` (abertura autoatendimento sentido único, PoW de
abertura minerado para você), `scone tld transfer <tld> <owner>`,
`scone tld revoke <tld>`, `scone domain assign <name> <owner>`
(owner do TLD, TLD privado) e `scone domain transfer <name>
<owner>`. Cada um: leitura de estado, assinatura, submissão, txid. A
resolução de registros DNS mudou para `scone domain resolve <name>`
(ex-`record get`); o canal de publicação hex (`record put`) sumiu —
a edição passa por `domain set` / `domain update`.
- `scone domain mine <name>` — mina APENAS a prova de registro
  (sem identidade, sem keystore, sem submissão): imprime o JSON
  `scone-domain-pow-v1` `{type, name, network_id, epoch, proof}`
  no stdout. Mineração delegada: outra máquina mina para você,
  suas chaves nunca saem da sua. Encaminhe por pipe ou redirecione:
  `scone domain mine n.tld | scone domain register n.tld
  --proof -` (ou `--proof FILE`). A prova está ligada ao nome e à
  rede; a frescura é imposta pelo nó na aplicação (janela de
  validade de época). Com `--mine-for <pubkey>`, ele mina em vez
  disso uma prova POW-TO-OWNER para esse beneficiário e imprime o
  JSON `scone-domain-pow-to-v1` `{type, name, network_id, epoch,
  pow_to, proof}` — consumível APENAS por `register --to` /
  `renew --to` para exatamente essa chave (o challenge a
  incorpora; verificação insensível a maiúsculas/minúsculas na
  submissão).
- Submissão pow-to-owner (sem necessidade de keystore — `--to` é
  exclusivo com `--identity`): `scone domain register <name>
  --to <pubkey>` / `scone domain renew <name> --to <pubkey>` e
  `scone tld register <tld> --to <pubkey>` / `scone tld renew
  <tld> --to <pubkey>` submetem uma prova cujo challenge liga a
  chave do BENEFICIÁRIO; o `owner` on-chain é recalculado como
  `hash(pubkey_dest)` (um renew NUNCA muda de mãos). Sem
  `--proof`, o comando mina ele mesmo a prova; com `--proof
  -`/FILE ele consome um payload `scone-domain-pow-to-v1` e
  recusa uma prova minada para OUTRO beneficiário. Estilo pipe:
  `scone domain mine n.tld --mine-for <pubkey> | scone domain
  register n.tld --to <pubkey> --proof -`.
- `scone domain export <name> [--out FILE]` — escreve o
  conjunto de registros verificado pela chain no formato exato de
  `update --file` (round-trip garantido: export → editar → update
  reenvia o mesmo conjunto). Sem `--out`: stdout. `domain resolve` usa o
  mesmo formato legível, portanto `resolve > zone.txt` é diretamente
  importável. Tipos sem sintaxe textual (GEOROUTE, desconhecidos)
  são exportados como avisos comentados — nunca silenciosamente
  descartados.

`domain get` também imprime o `domain_id` e `valid_until`.

## Benchmark de PoW (`scone bench-pow`)

Mede o hash-rate Argon2id da máquina (o mesmo hash que a
verificação de consenso usa) e traduz as dificuldades da rede em
tempos de espera concretos:

```
scone bench-pow --network mainnet [--seconds 5] [--cores N]
```

Saída: H/s total e por worker, e depois o tempo médio esperado por
operação — transferência/assign (taxa plana), abertura de um TLD
privado, e os pisos de registro TLD/domínio (a curva de
comprimento). A rede escolhida determina os parâmetros Argon2id
(mainnet: 64 MiB, t=2 — o custo real; testnet/devnet: o perfil
leve, provas em milissegundos). Cada worker aloca sua própria
lane Argon2id de 64 MiB — dimensione `--cores` pela RAM
disponível.

## Builds: dev vs mainnet-enabled (mainnet-compile-gate)

Os builds padrão **não carregam mainnet**: `scone relay --network
mainnet` (e serviços registradores com `SCONE_NETWORK=mainnet`)
falham imediatamente com uma dica de rebuild. O binário oficial
distribuído é construído com:

```bash
cargo build --release --features mainnet
```

`scone --version` mostra `+mainnet` nesses builds, e o
log de partida do relay diz `build: mainnet-enabled`.

O `--network` padrão segue o build também: `mainnet` em
builds mainnet-enabled (o binário oficial coloca os usuários na rede
de produção), `testnet` caso contrário. Uma flag `--network` explícita
ou o env `SCONE_NETWORK` sempre vencem
(ordem clap: flag > env > padrão). Isto é um
guardrail operacional (execuções acidentais, binários dev/CI) — não
uma fronteira criptográfica; as regras de consenso permanecem a defesa
contra nós hostis.
