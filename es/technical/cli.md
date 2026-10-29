# CLI (`scone`) — logging y verbosidad

> Comportamiento normativo del logging del binario `scone`. Cualquier
> divergencia doc/código = bug (corregir la fuente de verdad).

## Principios

- **stdout está reservado para la salida de comandos** (`scone status`,
  `identity list`, `status`, …). Un script puede redirigir stdout sin
  encontrar nunca logs en él — el piping sigue siendo fiable.
  Excepción aceptada: `scone relay` emite las líneas de datos
  `p2p: <multiaddr>`, `dns: <addr>` y `metrics: <addr>` en stdout
  tan pronto como los listeners están vinculados
  (datos utilizables para `--bootstrap` / configuración, no logs;
  emitidas exactamente una vez cada una, con flush inmediato).
- **Todos los logs van a stderr** (formato compacto de `tracing`),
  incluidos los del relay — el relay ya escribía a stderr antes de
  la migración.
- Los mensajes de error orientados al usuario del CLI
  (`scone: <error>`) se quedan en stderr: es UX, no logs.

## Niveles y `--verbose` / `-v`

| Invocación | Nivel efectivo |
|---|---|
| `scone <cmd>` (sin `-v`) | `warn` |
| `scone relay` (sin `-v`) | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …` (y más) | `trace` |

`-v`/`--verbose` es un flag **global y repetible**: se acepta antes
o después del subcomando (`scone -vv identity list` y
`scone identity list -vv` son equivalentes).

**Por qué `scone relay` se queda en INFO por defecto**: el relay es
un daemon — debe registrar su actividad (escucha RPC/p2p, bloques
aceptados y producidos, pares conectados, rechazos) para ser
operable, sin exigir un flag. Los comandos one-shot, en cambio, se
mantienen silenciosos (WARN) para no ensuciar una terminal o un
script.

## `RUST_LOG` (override)

Si la variable de entorno `RUST_LOG` está definida, **reemplaza
por completo** el nivel derivado de `-v` (sintaxis `env-filter`):

```bash
RUST_LOG=debug scone identity list        # debug sin -v
RUST_LOG=error scone -vv relay            # error gana a -vv
RUST_LOG=scone_network=trace,info scone relay   # por target
```

## Configuración por entorno (`.env`)

Antes de analizar los argumentos, `scone` carga un archivo `.env`
del directorio actual (`dotenvy::dotenv()` en `main.rs`). Archivo
ausente = no-op silencioso, nunca un error. Los valores cargados se
convierten en la línea base de las opciones del relay que declaran
una variable de entorno (`env = "…"` en `cli.rs`).

Precedencia (una opción, tres fuentes):

```text
flag CLI explícito  >  variable .env  >  valor por defecto integrado
```

- un flag pasado en la línea de comandos SIEMPRE gana;
- en caso contrario la variable (sea cargada desde `.env` o definida
  en el entorno real del proceso — misma variable, mismo efecto);
- en caso contrario el valor por defecto integrado de clap.

Las variables repetibles (`SCONE_BOOTSTRAP`, `SCONE_DNS_UPSTREAM`)
se acumulan: una línea por valor en `.env` (dotenvy añade).

La tabla completa de las variables `SCONE_*` (red, escucha P2P,
bootstrap, génesis, UPnP, superficie DNS, métricas, perfil de
topología, anchor) y la semántica de cada una vive en
[`relay.md`](relay.md#environment-configuration-env); la plantilla
comentada es `.env.example` en la raíz del repositorio (copiar a
`.env`, nunca hacer commit — `.env` está en gitignore).

## Semántica de niveles (convención del código)

- `error!`: fallos (hoy sin puntos de llamada — los fallos de red se
  tragan como `warn!` y solo los fallos locales fatales afloran
  como error tipado desde `run()`);
- `warn!`: entradas de red hostiles o maliciosas rechazadas, txs
  obsoletas desalojadas del mempool, txs conflictivas descartadas
  durante la producción, fallos salientes/entrantes,
  desajuste de claves DHT;
- `info!`: bloque aceptado/producido, escucha RPC/p2p, par
  conectado, arranque del relay (checkpoints de estado);
- `debug!`: detalles internos (tx aceptada, par desconectado, round
  trip RPC, listado del keystore, eventos kad bajo
  `SCONE_KAD_DEBUG`);
- `trace!`: reservado para payloads (hoy sin puntos de llamada).

Los crates de biblioteca (`scone-network`, …) emiten macros de
`tracing` pero **no inicializan ningún subscriber**: la
inicialización vive solo en el binario `scone` (antes del dispatch
del subcomando — el relay necesita sus logs desde el arranque).


## Uso simple (cli-simplify-v1)

- **Sin `--peer`**: los comandos hablan por defecto con el relay
  local; el env `SCONE_PEER` o `--peer` lo sobreescribe.
- **Sin `--identity`** cuando el keystore contiene una sola
  identidad (se usa en silencio; si hay varias → el error las
  lista).
- Eliminados (los stubs sugieren el reemplazo): `show`, `submit`,
  `lookup`, `owned`.
- Nuevos: `scone domain renew <name>`, `scone domains`, `scone
  domain get|set` (registro único, sin archivo).

```bash
scone identity generate me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone domain renew example.scone
```

## Ciclo de vida de los TLDs (cli-unified-lifecycle)

`scone tld register <tld>` es el punto de entrada de TLD en un solo
comando, simétrico con `scone domain register` — el estado on-chain
se lee PRIMERO, luego se ejecuta la acción correcta:

- **TLD libre** → se registra, naciendo **ABIERTO por defecto**
  (sin flag: espacio de nombres en autoservicio, 1/3 del PoW).
  `--private` opta por un espacio de nombres propio al precio del
  PoW completo, con confirmación explícita (decisión económica).
  `--yes` salta las confirmaciones (scripts).
- **tuyo, expira en menos de 90 días** → se propone la RENOVACIÓN
  (PoW ligada a la expiración actual — anti-reutilización) y se
  ejecuta tras la confirmación.
- **tuyo, lejos de expirar** → nada que hacer (`tld renew --force`
  para renovar de todos modos).
- **de otro** → error tipado limpio con el propietario.

```bash
scone tld register mytld            # nacimiento abierto (por defecto)
scone tld register mytld --private  # espacio de nombres propio (confirmar)
scone tld renew mytld               # renovación explícita
```

Igualmente, `scone domain register <name>` sobre un nombre tuyo que
expira en menos de 90 días propone la renovación en lugar de
fallar.

## Delegados de records (KSK/ZSK-lite)

El propietario de un dominio puede conceder a OTRAS claves el
derecho de firmar `domain update` en su nombre — el conjunto de
delegados de records (máximo 3 clés, reemplazo total en cada `set`):

```bash
# concesión (firmada por el propietario; el CLI recupera la
# generación CAS del namespace y liga la transacción a ella):
scone domain delegates set example.uip \
  --keys <pk-hex-1>,<pk-hex-2> --identity alice

# consulta (sin identidad):
scone domain delegates list example.uip

# revocar todo:
scone domain delegates clear example.uip --identity alice
```

Un delegado firma luego `domain update` exactamente como lo haría
el propietario (`--identity deleg`) — on-chain el campo `owner` de
la tx es el propietario del DOMINIO mientras que la firma viene del
delegado. La concesión es exclusiva del propietario (un delegado no
puede gestionar el conjunto), una transferencia la vacía, y cada
`set`/`clear` incrementa el contador de mutaciones del namespace
(CAS anti-replay — ver `transactions.md`): un `set` rejugado no
puede resucitar clés revocadas. La update de un delegado revocado
se rechaza localmente («neither the owner nor a record delegate»).

## Comandos avanzados

Los valores por defecto anteriores cubren el uso cotidiano. Para
scripts e interoperabilidad web/API la superficie completa sigue
disponible:

cada acción tiene exactamente UNA ruta canónica; los antiguos
alias de raíz (`show`, `lookup`, `owned`, `renew`) fueron
eliminados (remove-cli-aliases), igual que `domain info`, que
nunca correspondió a un comando real.

La superficie hex offline (subcomandos `tx build|sign|verify`,
`submit tx`) fue ELIMINADA (remove-tx-surface-v2): las operaciones
M8b que servía son ahora comandos de primera clase —
`scone tld open <tld>` (apertura unilateral en autoservicio, el PoW
de apertura se mina por ti), `scone tld transfer <tld> <owner>`,
`scone tld revoke <tld>`, `scone domain assign <name> <owner>`
(propietario del TLD, TLD privativo) y `scone domain transfer
<name> <owner>`. Cada uno: lectura de estado, firma, envío, txid.
La resolución de registros DNS pasó a `scone domain resolve <name>`
(ex-`record get`); el canal de publicación hex (`record put`)
desapareció — la edición pasa por `domain set` / `domain update`.
- `scone domain mine <name>` — mina SOLO la prueba de registro
  (sin identidad, sin keystore, sin envío): imprime el JSON
  `scone-domain-pow-v1` `{type, name, network_id, epoch, proof}`
  a stdout. Minado delegado: otra máquina mina por ti, tus claves
  nunca salen de la tuya. Encamínalo o redirígelo:
  `scone domain mine n.tld | scone domain register n.tld
  --proof -` (o `--proof FILE`). La prueba está ligada al nombre y
  a la red; la frescura la hace cumplir el nodo al aplicar
  (ventana de validez de época). Con `--mine-for <pubkey>` mina en
  su lugar una prueba POW-TO-OWNER para ese beneficiario e imprime
  el JSON `scone-domain-pow-to-v1` `{type, name, network_id,
  epoch, pow_to, proof}` — consumible SOLO por `register --to` /
  `renew --to` para exactamente esa clave (el challenge la
  incorpora; comprobación insensible a mayúsculas/minúsculas en el
  envío).
- Envío pow-to-owner (sin necesidad de keystore — `--to` es
  exclusivo con `--identity`): `scone domain register <name>
  --to <pubkey>` / `scone domain renew <name> --to <pubkey>` y
  `scone tld register <tld> --to <pubkey>` / `scone tld renew
  <tld> --to <pubkey>` envían una prueba cuyo challenge liga la
  clave del BENEFICIARIO; el `owner` on-chain se recalcula como
  `hash(pubkey_dest)` (un renew NUNCA cambia de manos). Sin
  `--proof`, el comando mina él mismo la prueba; con `--proof
  -`/FILE consume un payload `scone-domain-pow-to-v1` y rechaza
  una prueba minada para OTRO beneficiario. Estilo pipe:
  `scone domain mine n.tld --mine-for <pubkey> | scone domain
  register n.tld --to <pubkey> --proof -`.
- `scone domain export <name> [--out FILE]` — escribe el conjunto
  de registros verificados contra la cadena en el formato exacto de
  `update --file` (round-trip garantizado: export → editar →
  update reenvía el mismo conjunto). Sin `--out`: stdout.
  `domain resolve` usa el mismo formato legible, así que
  `resolve > zone.txt` es directamente importable. Los kinds sin
  sintaxis textual (GEOROUTE, desconocidos) se exportan como
  advertencias comentadas — nunca se descartan en silencio.

`domain get` también imprime el `domain_id` y `valid_until`.

## Benchmark de PoW (`scone bench-pow`)

Mide el hash-rate Argon2id de la máquina (el mismo hash que usa la
verificación del consenso) y traduce las dificultades de la red en
tiempos de espera concretos:

```
scone bench-pow --network mainnet [--seconds 5] [--cores N]
```

Salida: H/s total y por worker, y después el tiempo medio esperado
por operación — transferencia/assign (tasa plana), apertura de un
TLD privativo, y los suelos de registro TLD/dominio (la curva de
longitud). La red elegida determina los parámetros Argon2id
(mainnet: 64 MiB, t=2 — el coste real; testnet/devnet: el perfil
ligero, pruebas en milisegundos). Cada worker asigna su propio
carril Argon2id de 64 MiB — dimensiona `--cores` según la RAM
disponible.

## Builds: dev vs mainnet habilitada (mainnet-compile-gate)

Los builds por defecto **no llevan mainnet**: `scone relay --network
mainnet` (y los servicios del registrar con `SCONE_NETWORK=mainnet`)
fallan inmediatamente con una pista de recompilación. El binario
oficial distribuido se construye con:

```bash
cargo build --release --features mainnet
```

`scone --version` muestra `+mainnet` en esos builds, y el log de
arranque del relay dice `build: mainnet-enabled`.

El `--network` por defecto también sigue al build: `mainnet` en
builds con mainnet habilitada (el binario oficial aterriza a los
usuarios en la red de producción), `testnet` en caso contrario. Un
flag `--network` explícito o el env `SCONE_NETWORK` siempre gana
(orden clap: flag > env > por defecto). Es una barrera operativa
(arranques accidentales, binarios dev/CI) — no una frontera
criptográfica; las reglas de consenso siguen siendo la defensa
contra nodos hostiles.
