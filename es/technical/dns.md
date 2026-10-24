# Servidor DNS UDP + TCP (M6 + M7b) — `scone dns`

> Especificación normativa de la superficie DNS de Scone.
> Cualquier divergencia doc/código = bug (corregir la fuente de
> verdad).

## Visión general

El **servidor DNS** es una superficie opcional del relay: responde
consultas DNS **UDP y TCP** (RFC 7766, mismo puerto) para los
nombres Scone **solo a partir de datos verificados contra la
cadena**, y ofrece un **fallback recursivo opcional**
(`addr:port`) para todo lo demás. No posee ni cadena, ni almacén,
ni DHT: las decisiones de existencia/expiración pasan por el
**snapshot DNS autoritativo** (una vista RAM publicada tras cada
bloque aplicado — ver abajo), y las recuperaciones de registros
pasan por el relay (proceso), vía el canal interno
`RelayCommand`/`ResolveLocal` (en-proceso, un round trip) — la
misma ruta verificada que el CLI.

```text
              UDP+TCP :5353 (por defecto 127.0.0.1)
                    │
             scone relay --dns 127.0.0.1:5353
                    │  ┌────────────────────────────────┐
                    ├──┤ Caché por shards, 16 vías (qname)│
                    │  └────────────────────────────────┘
                    │  ¿Nombre con forma Scone?
                    │  ├─ SÍ → snapshot DNS autoritativo:
                    │  │        desconocido/caducado → NXDOMAIN (RAM,
                    │  │        cero round trips al core)
                    │  │        → búsqueda de ápice por sufijo
                    │  │        → `resolve_local` interno (relay)
                    │  │           cadena + registro verificado (hash,
                    │  │           owner, secuencia) → respuestas
                    │  │        ápice sin registro publicado → NODATA
                    │  └─ NO → fallback upstream (si configurado)
                    │           si no REFUSED
                    │
                    │  presupuesto de respuesta:
                    │    UDP → 512 bytes (sin EDNS); desbordamiento =
                    │          corte en el último registro que quepa + TC=1 →
                    │          el cliente reintenta por TCP
                    │    TCP → 64 KiB (prefijo de longitud de 2 bytes,
                    │          RFC 1035 §4.2.2): conjunto completo
```

## Arranque

```bash
scone relay --dns 127.0.0.1:5353            # solo superficie DNS
scone relay --dns 0.0.0.0:53 \
            --dns-upstream 1.1.1.1:53 \
            --dns-upstream 8.8.8.8:53       # + fallback recursivo
```

El relay anuncia la dirección efectiva en stdout (contrato
legible por máquina, como `p2p:`): una línea `dns: <addr>` tan
pronto como el socket está vinculado.

Cliente de validación: `dig <name> --dns 127.0.0.1:5353
--qtype A` (A, AAAA, TXT, MX, NS, CNAME, ANY).

## Ruta de resolución (AUTORIDAD = la cadena, siempre)

0. **Snapshot DNS autoritativo**: una vista inmutable
   (`Arc<HashMap<DomainId, {valid_until, has_record}>>`,
   feed `tokio::sync::watch`) publicada por el núcleo del relay
   **tras cada bloque aplicado** (push, no poll) y al arrancar.
   Las decisiones de existencia/expiración — NXDOMAIN, dominio
   caducado — se sirven desde RAM a través de este snapshot: un
   clon de `Arc` + una búsqueda en el mapa, **sin entrar jamás en
   el run loop monopropietario del relay**. La ruta completa
   (`RelayCommand`/`ResolveLocal`) queda reservada para las
   recuperaciones reales de registros (pulls DHT), raras gracias a
   la caché. El snapshot NUNCA lleva datos de registro: solo
   existencia y expiración.
   - **Deltas copy-on-write O(tocados)**: el mapa es un
     `Arc<HashMap>`; una extensión del tip publica el delta del
     bloque (las propias transacciones del bloque nombran cada
     dominio que pueden cambiar — Register/Assign/Update/Renew/
     TransferDomain — más las eliminaciones deterministas del
     GC de expiración evaluadas en el push) mediante
     `Arc::make_mut`: el mapa se clona SOLO si un lector conserva
     aún la vista anterior. El arranque y los reorgs reconstruyen
     el snapshot completo (un paseo paginado del backend).
   - Frescura: un lector puede ver un snapshot con unos cientos de
     ms de retraso durante la aplicación de un bloque — la misma
     obsolescencia que tendría un RPC de estado; los TTL de la
     caché DNS no se ven afectados (las respuestas siguen siendo
     autoritativas según su origen).
1. **Caché por shards**: [`ShardedCache`] — 16 shards `Cache`
   independientes con clave `hash(qname)` (el qtype comparte el
   shard: localidad por nombre). Un nombre caliente compite con
   1/16 del tráfico en lugar de hacer cola tras un único mutex a
   miles de qps. Cotas GLOBALES (100.000 entradas / 64 MiB por
   defecto, `--dns-cache-limit` / `--dns-cache-max-mem`)
   repartidas por shard (peor caso documentado: un factor
   1 + N/shards). El tamaño REAL del rdata de cada entrada se
   contabiliza (~128 bytes de sobrecoste por entrada) y el
   desalojo FIFO se dispara con la primera de las dos cotas
   alcanzadas — el tope de memoria se mantiene incluso contra
   entradas de tamaño completo (`MAX_DHT_RECORD_LEN`). TTL 60 s
   (autoritativo) / recortado a 600 s (fallback). La caché
   negativa (NXDOMAIN/NODATA) sigue las mismas reglas; cada
   entrada lleva su origen (autoritativo o fallback) para que el
   bit AA se mantenga honesto en un hit. **Nunca cachear
   SERVFAIL** (fallo local transitorio) **ni REFUSED** (estado de
   configuración): ambos se recomputan en cada consulta.
   Desalojo FIFO a capacidad. **Ruta caliente (inspirada en
   Cloudflare/1.1.1.1)**:
   - entradas `Arc<CacheEntry>` compartidas: un hit es un bump de
     refcount, NUNCA un clon profundo del rdata (`put` REEMPLAZA
     el Arc — sin mutación compartida; el TTL se comprueba igual
     bajo el bloqueo del shard);
   - **búsqueda sin asignaciones**: mapa de dos niveles
     (`Box<str>` qname → qtype → entrada), el `&str` prestado de
     la consulta sondea directamente (`Borrow<str>`) — la antigua
     clave tupla `(String, u16)` asignaba un `String` fresco en
     CADA búsqueda;
   - **registros contiguos empaquetados**: un búfer `Box<[u8]>`
     por entrada (`u16 rdlen ‖ u16 rtype ‖ u32 ttl ‖ rdata` por
     registro) en lugar de un `Vec` por registro — el presupuesto
     de memoria sigue exactamente lo asignado;
   - **compresión de nombres RFC 1035 en la salida**: el owner de
     las respuestas autoritativas es un puntero de compresión de
     2 bytes (`0xC00C`) hacia la pregunta repetida — la pregunta
     nunca se re-codifica completa por registro.
2. **¿Forma Scone?** Pre-filtro estricto: etiquetas `[a-z0-9-_]`
   (1..63), ≥ 2 etiquetas, TLD de 1..63 bytes `[a-z0-9-]` (ver
   `TldName::MAX_LEN`), longitud total ≤ 253 — **y TLD fuera de
   la raíz ICANN** (`is_icann_tld`, lista IANA congelada + RFC
   2606: `com`, `fr`, `org`… nunca pertenecen a Scone).
   Un nombre que NO PUEDE ser un nombre Scone (p. ej.
   `www.foo_bar`, guion bajo en el TLD) va al fallback; un
   posible nombre Scone (`absent.uip`) es asunto de la cadena:
   sin registrar → NXDOMAIN autoritativo, incluso sin upstream.
   Un TLD ICANN (`example.com`) se reenvía a los upstreams si
   están configurados, si no REFUSED — nunca resuelto
   autoritativamente por Scone.
3. **Búsqueda de ápice**: sufijos del más largo al más corto (mín.
   2 etiquetas). El primer sufijo cuyo estado on-chain es conocido
   por el relay = ápice; el conjunto de registros sirve al qname
   entero (subnombres incluidos — comportamiento comodín
   implícito, decisión abierta).
4. **`resolve_local` (canal interno)**: un solo round trip, nunca
   un waiter DHT. Estado on-chain (`registered`, owner, secuencia)
   + registro de la caché DHT local **re-verificado** (owner,
   secuencia, record_hash == compromiso on-chain). Un registro no
   conforme nunca responde — disponibilidad ≠ autoridad.
   - sin registrar → NXDOMAIN;
   - registrado sin registro válido → NODATA (NOERROR, 0
     respuestas);
   - registro verificado → respuestas filtradas por qtype.

## Códec (RFC 1035, subconjunto estricto)

- Consulta: cabecera de 12 bytes, **una** pregunta, QCLASS = IN,
  **sin compresión** (puntero = rechazar). Etiquetas en minúsculas
  y validadas. QR=1, QDCOUNT≠1, etiquetas inválidas →
  **silencio** (la basura no cuesta nada).
- Respuesta: QR|RD(eco)|RA, **AA solo en respuestas
  autoritativas** (cadena, frescas o en caché — nunca en fallback
  ni en RCODEs de error), pregunta copiada verbatim, respuestas
  autoritativas con **owner comprimido RFC 1035** (puntero de 2
  bytes `0xC00C` hacia la pregunta repetida); las respuestas del
  upstream de fallback se sirven tal cual (nunca re-codificadas).
- Tipos servidos: A, AAAA, CNAME, NS, MX, TXT, **PTR, SRV, CAA**
  (+ Unknown re-codificado verbatim; 9 tipos nativos en total,
  rdata codificado por RFC — 1035/2782/8659; CAA estricto: bit
  crítico 0x80 únicamente, etiquetas en minúsculas). Una consulta
  A/AAAA incluye los CNAMEs del conjunto (persecución de alias).
  Consulta ANY (255): el conjunto entero filtrado por el códec.
- Límites (M7b): **el presupuesto de respuesta depende del
  transporte** — 512 bytes por UDP (payload clásico RFC 1035
  §2.3.4, sin EDNS), 65.535 por TCP. Un conjunto que exceda el
  presupuesto se corta en el último registro que quepa, con
  **TC=1** (truncamiento honesto, RFC 1035 §4.2.1): el cliente
  reintenta por TCP (RFC 7766), donde se sirve el conjunto
  completo. Una respuesta del upstream mayor que el presupuesto
  UDP se re-codifica truncada (TC=1) en lugar de entregarse tal
  cual; por TCP pasa byte a byte. El truncamiento aplica también a
  los hits de caché (el presupuesto se aplica en el momento de la
  respuesta, no al llenar la caché).
- RCODEs: 0 NOERROR (con o sin respuestas), 2 SERVFAIL (fallo
  local), 3 NXDOMAIN (autoritativo), 5 REFUSED (no-Scone sin
  upstream).

## Fallback recursivo

- Upstreams `addr:port` (UDP), probados en orden, 2 s de timeout
  por upstream, socket **conectado** al upstream (filtrado por el
  kernel: un paquete de cualquier otra fuente jamás se acepta).
  La respuesta del upstream solo se acepta si **coincide con la
  consulta**: mismo TXID, QR=1, pregunta (QNAME+QTYPE+QCLASS)
  copiada byte a byte. Una respuesta no conforme → SERVFAIL, y no
  se cachea nada.
- La respuesta del upstream validada se pasa tal cual; sus
  respuestas se cachean con origen `fallback` (AA nunca activado,
  incluso en un hit de caché) y TTL recortado a 600 s. Los RCODEs
  de error del upstream (distintos de NOERROR/NXDOMAIN) no se
  cachean.
- **Privado por defecto**: sin upstream = REFUSED para los nombres
  fuera de la forma Scone.

## Caché negativa DHT (anti-amplificación)

Un **pull DHT sin respuesta** (registro esperado on-chain —
`record_hash` comprometido — pero ausente de la caché local y de
los pares consultados) aparca el nombre durante
`DHT_NEGATIVE_TTL_SECS` (30 s) en una caché negativa acotada
(FIFO, tope `DHT_NEGATIVE_CAP` = 4.096 nombres): las
resoluciones repetidas del mismo nombre dentro del TTL quedan
**suprimidas** (contadas, nunca re-pulleadas). Un escáner — o un
nombre caliente cuyo propietario nunca publicó — ya no puede
amplificar una consulta DNS en un `GetRecord` por par de la malla
por consulta. La entrada negativa muere al TTL, a la llegada del
registro (el pull exitoso escribe en la caché real), o al ser
empujada fuera del tope FIFO.

Contadores expuestos (`scone_dht_negative_hits_total`,
`_expired_total`, `_capped_total` — ver «Métricas» en
`relay.md`).

## TCP (M7b, RFC 7766)

Mismo puerto que UDP (segundo bind TCP en la dirección `--dns`). El
transporte sigue RFC 1035 §4.2.2: cada mensaje va prefijado por su
longitud en **2 bytes** big-endian.

- **Consulta**: longitud del prefijo ≥ 1 y ≤ 512 bytes (una
  consulta DNS cabe en el payload clásico — RFC 7766 §6; sin EDNS
  servido). Fuera de cotas → cierre inmediato de la conexión.
- **Respuesta**: presupuesto de 65.535 bytes (el máximo del
  prefijo de 2 bytes) — un conjunto demasiado grande para UDP se
  sirve aquí **completo**, TC=0.
- **Pipelining**: múltiples consultas por conexión se sirven en
  serie, en orden (una escritura = una respuesta prefijada).
- **Timeout de lectura**: 5 s por lectura (prefijo o cuerpo) — un
  cliente silencioso se desconecta, su slot se libera.
- **Conexiones concurrentes**: 64 máximo (semáforo); la
  65.ª se rechaza inmediatamente (conexión cerrada), nunca en
  cola — sin crecimiento no acotado de tareas o memoria.
- Errores de lectura/escritura/timeout: log debug + cierre limpio.
  **Nunca panic con entrada de red.**
- La basura (consulta incomprensible) recibe silencio pero no
  cierra la conexión: la siguiente consulta (válida) se sirve.

```text
cliente                    servidor
  │ [len u16][consulta]     │
  │ ─────────────────────►  │ caché → autoritativo → fallback
  │ ◄─────────────────────  │ [len u16][respuesta ≤ 64 KiB]
  │ (UDP había respondido TC=1 → el cliente reintenta aquí)
```

## Cotas y barreras

| Barrera | Valor |
|---|---|
| Datagrama UDP (consulta/respuesta) | 4096 bytes (rechazado más allá) |
| Presupuesto respuesta UDP (TC=1 más allá) | 512 bytes (sin EDNS) |
| Consulta TCP máxima | 512 bytes (cierre más allá) |
| Respuesta TCP máxima | 65.535 bytes |
| Conexiones TCP concurrentes | 64 (semáforo, rechazo inmediato) |
| Timeout de lectura TCP | 5 s |
| Caché | cotas GLOBALES 100.000 entradas / 64 MiB por defecto, repartidas en 16 shards (`--dns-cache-limit` / `--dns-cache-max-mem`) |
| TTL autoritativo | **por registro** (`record.ttl`, cotas 1..=2.592.000 s; la constante histórica de 60 s queda solo para respuestas de servicio) |
| TTL fallback (recorte) | 600 s |
| Consultas UDP en paralelo | 4.096 (`MAX_INFLIGHT_UDP`, semáforo; más allá: descarte, el cliente reintenta) — ajustable `--dns-max-inflight` (mín 1) |
| Caché negativa DHT | 30 s (`DHT_NEGATIVE_TTL_SECS`), tope FIFO 4.096 nombres |
| Timeout por upstream | 2 s |

Reglas del proyecto respetadas: nunca panic con datos de red;
decodificación estricta en todas partes; el servidor no toca ni
cadena, ni almacén, ni DHT (el relay sigue siendo la única
autoridad de verificación).

## Tests

- Unitarios (`scone-network::dns`): códec estricto (QR, compresión,
  rechazos de etiquetas), respuestas autoritativas, NODATA,
  NXDOMAIN, REFUSED sin upstream, fallback contra un mock UDP,
  caché acotada, troceo TXT, análisis del upstream; corrección M6:
  honestidad del bit AA (autoritativo vs fallback vs errores,
  incluidos hits de caché), truncamiento TC=1 de un conjunto
  sobredimensionado, rechazo de respuesta del upstream (TXID o
  pregunta no coincidentes), no cachear SERVFAIL/REFUSED; caché
  por shards: reparto de shards, cotas globales divididas, 4
  trabajadores concurrentes sobre un nombre caliente; ruta
  caliente: búsqueda sin asignaciones, entradas empaquetadas,
  compresión del owner.
- M7b: presupuesto por transporte (UDP 512 bytes → TC=1, TCP →
  conjunto completo), truncamiento de un hit de caché, `run_tcp`
  real (conjunto > 512 servido completo, pipelining de 2 consultas,
  consulta sobredimensionada → cierre, cliente silencioso →
  timeout de 5 s, basura y luego consulta válida en la misma
  conexión), respuesta del upstream > 512 re-codificada TC=1 por
  UDP / byte a byte por TCP.
- Integración real (`tests/dns_server.rs`): un relay completo
  (incluida producción devnet) + registro + UpdateDomain +
  PutRecord, luego consultas UDP reales: A verificado (bit AA),
  TXT, NODATA AAAA, NXDOMAIN, fallback vía mock, re-consulta,
  subnombre vía ápice.
- E2E binario (`crates/scone/tests/dns_cli.rs`): el binario `scone`
  real — `relay --dns`, `identity generate`, `domain register`,
  `domain update --file`, luego `dig` (dnsutils) (A verificado,
  TXT, NXDOMAIN, REFUSED) en un puerto UDP real.

## Decisiones abiertas

- Sin EDNS (el presupuesto UDP se queda en 512 bytes; el
  truncamiento TC=1 + TCP cubre la necesidad devnet).
- El conjunto de registros no lleva nombre por registro: el
  conjunto entero sirve a cada subnombre del ápice (comodín
  implícito). El direccionamiento por registro exigiría extender
  `DnsRecord` (protocolo).
- El fallback reenvía la consulta cruda: un upstream malicioso
  puede responder cualquier cosa (respuesta no validada — este es
  el modelo recursivo estándar; los nombres Scone NUNCA pasan por
  él).

## GeoDNS (enrutado geográfico)

> Especificación normativa. Cualquier divergencia doc/código =
> bug.

### Tabla GeoIP embebida (automática)

Cada binario embebe una **tabla de mapeo IP → país** compilada en
el relay: **sin configuración necesaria**, el enrutado geográfico
está activo en todas partes, automáticamente.

- **Fuente de datos**: **DB-IP Lite** («IP to Country Lite»),
  con licencia **CC BY 4.0** — redistribuible con atribución:
  `https://db-ip.com` (curada mensualmente; antes: las
  delegaciones de los RIR vía `scripts/gen-geo-table.py`,
  retirado). Celdas cuantizadas **/20 v4 y /24 v6** (la
  granularidad nativa de DB-IP, ~1,3 M de rangos, supera el límite
  de memoria del parser) — ~96.500 reglas. La fecha de generación
  está en la cabecera de la tabla
  (`assets/embedded-geo.table`).
- **Checksum**: el SHA-256 de la tabla se calcula en tiempo de
  compilación (`build.rs`), **se reverifica en el arranque**
  (mismatch = rechazo tipado — un binario alterado nunca sirve
  respuestas geográficas) y se registra en el arranque con la
  atribución. El hash garantiza que «todos los nodos de un mismo
  build comparten la misma tabla» es COMPROBABLE, no asumido.
- **Regeneración**: `scripts/geo-convert.py` (descarga el CSV de
  DB-IP Lite del mes, convierte — país numérico, continente
  derivado ISO→M49, ASN/región ausentes de la Lite).
- **Granularidad**: país (ISO 3166-1 numérico) + continente (ONU
  M49) derivado. Sin ASN ni región (ausentes de la Lite — una
  necesidad de ASN/región espera una evolución del pipeline
  embebido, nunca una tabla local).
- **Rendimiento**: búsqueda por **búsqueda binaria** sobre rangos
  ordenados (índice construido una vez en el primer uso, por
  familia v4/v6) — medido ~95 ns/búsqueda en caliente, ~50 µs en
  acceso aleatorio sobre la tabla completa (bench: `cargo test -p
  scone-network --release --test geo_bench -- --ignored`).
- **Auto-localización del relay**: el enrutado sigue la **IP de
  origen de cada consulta** — un relay de LAN dirige a sus
  clientes locales a la zona correcta automáticamente.
- **Tabla IMPUESTA** (remove-geo-operator-table, 2026-10-22): la
  tabla embebida es la **única fuente** — no existe import ni
  sobreescritura por parte del operador. **No existe ninguna opción
  geo en tiempo de ejecución**: todo flag `--geo-*` es un rechazo
  clap seco (la geozona la impone el build). Un operador con
  necesidades específicas (ASN, región) espera una evolución del
  pipeline embebido, no una tabla local.

### Visión general

Un dominio puede publicar reglas de **enrutado geográfico** en su
conjunto de registros firmados: registros `GEOROUTE` (tipo
privado de Scone `32.768`, fuera del rango IANA). El resolvedor
responde a las preguntas A/AAAA de estos dominios con **la
dirección del endpoint elegido por la regla** (ámbito más próximo
a la posición del CLIENTE, derivada de su IP de origen vía la
tabla), en lugar de un registro A/AAAA estático. Todo lo demás
(dominios sin GEOROUTE, otros qtypes) sigue la ruta DNS clásica,
**byte a byte**.

### Métricas GeoDNS

El endpoint Prometheus expone el reparto de respuestas geo:

- `scone_dns_geo_answers_total` — total de respuestas GEOROUTE
  servidas;
- `scone_dns_geo_by_continent_total{continent="<M49>"}` — por
  continente;
- `scone_dns_geo_by_country_total{country="<ISO>"}` — por país
  (acotado por los propios estándares: ≤ 9 continentes, ≤ 250
  países).

### Registro GEOROUTE (wire)

Un GEOROUTE = un **ámbito** + ≤ 32 **endpoints** (orden canónico en
decodificación: prioridad ascendente, peso descendente, target
ascendente; duplicado = rechazar):

```text
GeoScope   = 0x00 (mundo)
           | 0x01 continent.v (código ONU M49: AF=2, AS=142, EU=150,
                                NA=3, SA=5, OC=9, AN=999)
           | 0x02 country.v   (código numérico ISO 3166-1: US=840,
                                TW=158, FR=250…)
           | 0x03 region.v    (subdivisión ISO 3166-2 OPACA, u32:
                                número definido por el publicador,
                                comparado tal cual — sin tabla local)
           | 0x04 asn.v       (número de AS, u32)
           | 0x05 prefix.addr[16] || prefix.len u8
             (prefijo IP normalizado de 16 bytes: IPv4 = dirección en
              los 4 primeros bytes + cola CERO, len ≤ 32;
              IPv6 = 16 bytes libres, len ≤ 128; los bits de host
              SIEMPRE a cero — un prefijo no normalizado se rechaza)
GeoTarget  = 0x00 octets[4]  (dirección IPv4 binaria)
           | 0x01 octets[16] (dirección IPv6 binaria)
GeoEndpoint = target || priority.v || weight.v
GEOROUTE    = scope || count.v(≤32) || endpoints…
```

Cotas de validación (decodificación): `priority > 0`,
`weight ≤ 100` (`weight = 0` = endpoint inactivo, nunca electo),
TTL dentro de 1..=2.592.000 s como cualquier registro. Un
país/continente desconocido para las tablas locales está
**permitido** en el wire: el enrutado simplemente cae a un ámbito
superior.

**Compatibilidad wire hacia atrás (v1 → v2)**: los bytes v1 quedan
SIN CAMBIOS — los ámbitos v1 (mundo/continente/país) conservan sus
discriminantes `0x00..=0x02` y su codificación exacta; los tres
nuevos ámbitos toman los discriminantes siguientes (`0x03..=0x05`)
en un orden total coherente con el orden canónico
(`World < Continent < Country < Region < Asn < IpPrefix`). Un
decodificador v1 que encontrara un discriminante ≥ `0x03` ya
fallaba estrictamente (registro rechazado); los bytes producidos
por v1 se siguen decodificando idénticamente (ver el test
`geo_known_bytes`, congelado).

### Selección (resolución)

1. **Ámbito** — especificidad decreciente, **6 niveles**:
   **prefijo IP más largo → AS exacto → país exacto →
   región exacta → continente → mundo**.
   - *IpPrefix*: entre las rutas cuyo prefijo coincide con la IP
     del cliente (tal como la conoce el localizador: `client_ip`,
     si no los prefijos de origen conocidos), el **más largo**
     (`len` máx) gana; a igual `len`, el primero en orden canónico
     (el orden del slice `routes`, ordenado en codificación). Una
     subdivisión es menos específica que un prefijo/AS (que
     identifican la red exacta) y menos que el país (numeración
     opaca, fiable solo si el publicador y el localizador
     comparten la misma), pero más que el continente (está
     contenida en su país, y por tanto a fortiori en su
     continente).
   - *Asn*: número de AS exacto del cliente.
   - *Country*: país exacto.
   - *Region*: subdivisión exacta (identificador opaco).
   - *Continent*: aportado por el localizador **o derivado del
     país** vía la tabla local (`continent_of_country`); un país
     desconocido para la tabla cae al continente explícito o al
     mundo.
   - *World*.
   Una [`GeoLocation`] v1 (region/asn/prefixes/client_ip sin
   fijar) produce exactamente el comportamiento v1: país →
   continente → mundo. Sin ruta coincidente (incluso sin ámbito
   World) → comportamiento DNS clásico, como si el GEOROUTE no
   existiera.
2. **Prioridad** — semántica tipo SRV: solo el grupo de **prioridad
   mínima** es elegible; las prioridades mayores son niveles de
   **failover**.
3. **Peso** — dentro del grupo electo, sorteo **ponderado
   determinista**: hash estable (SplitMix64) de un nonce (el
   contador de consultas del resolvedor) contra los pesos
   acumulados. Sin RNG: dos consultas con el mismo nonce dan el
   mismo resultado, los nonces crecientes reparten
   proporcionalmente a los pesos. `weight = 0` = inactivo, nunca
   electo (incluso solo — pasar al nivel siguiente).
4. **Orden firmado, cero sondeos** — el resolvedor NUNCA juzga la
   disponibilidad de un endpoint: todo endpoint activo (weight > 0)
   del ámbito electo es candidato, la cabeza del nivel mínimo se
   elige por el sorteo ponderado, y los siguientes actúan como
   failovers **en el orden del conjunto de registros firmado** (el
   orden ES el mecanismo de failover). La alcanzabilidad de un
   endpoint es asunto del operador de ese endpoint, nunca del
   resolvedor.

### Respuesta

- Una sola dirección: **A si IPv4, AAAA si IPv6** — filtrado
  estricto por familia según el qtype (una pregunta A NUNCA
  responde un IPv6, y viceversa).
- TTL = el del registro GEOROUTE de origen (acotado en
  decodificación por las reglas generales de TTL).
- AA activado (respuesta autoritativa — el GEOROUTE es un registro
  firmado verificado como cualquier otro).
- Respuesta vacía tras el filtrado (ningún candidato de la familia
  correcta, ninguna ruta para la zona) → el
  resolvedor continúa por su ruta clásica (típicamente NODATA:
  NOERROR, 0 respuestas); los rcodes de error siguen las reglas
  generales (nunca cacheados).
- Un registro A/AAAA clásico que coexista en el conjunto toma
  PRECEDENCIA sobre geo (geo aplica solo en ausencia de respuesta
  wire para el qtype).

### Caché: decisión explícita

La decisión de enrutado geo **NUNCA se cachea** bajo
`(qname, qtype)`: depende del ámbito del cliente del resolvedor, y
tal clave serviría la ruta de US a un cliente TW. Las respuestas
geo se recomputan en cada consulta (coste: un select_scope + sorteo
ponderado, µs) y saltan la caché. La caché clásica (qname, qtype)
permanece sin cambios para todo lo demás — incluido el NODATA
producido cuando geo no decide nada. Los rcodes de error siguen
sin cachearse nunca.

### Delegación NS (registros `ns`) — el owner elige quién sirve

El owner de un dominio elige el modo de servicio por dominio:

- **DHT (por defecto)** — sin registro NS, la red Scone sirve los
  A/AAAA on-chain (relay `--dns`, o el puente NSD vía export);
- **NS independiente** — el owner añade registros
  `ns <ip-o-nombre>` (SUS nameservers, IP directa o nombre de
  host): el dominio queda DELEGADO — la red deja de servir su
  contenido (A/AAAA → NODATA + autoridad NS), y quien resuelve
  sigue los NS del owner. La cota: **12 NS distintos por dominio**
  (todas las formas, deduplicados por valor — espejo de los
  límites de los registros reales). Retirar un NS libera el slot;
  retirar todos restaura el modo DHT.

**No existe ninguna comprobación de salud**: la disponibilidad de
los endpoints es asunto de los operadores de los endpoints — el
resolvedor sirve el conjunto firmado tal cual, en el orden de los
registros (el orden de los registros es el mecanismo de failover).

### Seguridad

- Los registros GEOROUTE son registros firmados como cualquier
  otro: verificados contra la cadena (owner, secuencia,
  record_hash) antes de cualquier uso. **Sin confianza de red sin
  validación** — un GEOROUTE DHT sin verificar nunca responde.
- La posición del cliente viene del **circuito autodescubierto**:
  el relay se autolocaliza vía su dirección externa (STUN/AutoNAT/
  UPnP) resuelta por la tabla embebida impuesta por el build — los
  clientes LAN (IP de origen privada) heredan esa posición, los
  clientes de Internet se resuelven por su propia IP, lo
  desconocido cae al mundo. Nunca de una API GeoIP de red (sin
  fuga de consultas, sin dependencia) y nunca de una declaración
  del operador (las geozonas las impone el build).
- El sorteo es determinista y local: sin secreto, sin
  imprevisibilidad requerida (la seguridad es la de la firma del
  registro, no la del sorteo).

### Ejemplo completo (especificación de referencia)

`www.example.scone` publica (ápice `example.scone`, el conjunto
entero sirve a cada subnombre):

| Ámbito | Endpoint | Prioridad | Peso |
|---|---|---|---|
| US (840) | 10.1.0.1 | 10 | 70 |
| US (840) | 10.1.0.2 | 10 | 30 |
| US (840) | 10.2.0.1 | 20 | 100 |
| TW (158) | 10.3.0.1 | 10 | 80 |
| TW (158) | 10.4.0.1 | 10 | 20 |
| TW (158) | 10.2.0.1 | 20 | 100 |
| WORLD | 10.0.0.1 | 100 | 100 |

Comportamiento esperado:

- cliente **US** → 10.1.0.1 (≈70%) o 10.1.0.2 (≈30%); el nivel p20
  solo se alcanza si ningún endpoint p10 está activo (weight = 0
  en todos) → entonces **10.2.0.1**;
- cliente **TW** → 10.3.0.1 (≈80%) o 10.4.0.1 (≈20%);
- cliente **FR** (250) → sin ruta FR ni EU en el conjunto → WORLD
  → **10.0.0.1**;
- pregunta **A** sobre una ruta solo IPv6 → respuesta vacía
  (nunca un IPv6 en A);
- dominio sin regla aplicable → misma pregunta, comportamiento
  clásico (NODATA: el GEOROUTE no tiene rdata A en el wire).

### Opciones CLI (`scone relay`)

| Opción | Por defecto | Rol |
|---|---|---|
| (ninguna — GeoDNS siempre activo, tabla embebida) | activo | Enrutado automático por IP de origen |
| `--geo-ip-table`, `--geo-config`, `--geo-country/…`, `--health-*` | — | (ELIMINADAS — rotura seca: la geozona la impone el build y el resolvedor no hace ninguna comprobación de salud; no existe ningún flag en tiempo de ejecución) |

#### La tabla embebida es la única fuente

La tabla GeoIP está **impuesta por el build** (DB-IP Lite,
cuantizada, ver la sección de la tabla embebida más arriba): no se
puede importar ninguna tabla de operador. Resolución: la **IP de
origen de la consulta DNS** (por cliente: UDP `recv_from` / TCP
`peer_addr`) se compara en 16 bytes contra la tabla embebida; la
regla con el **prefijo coincidente más largo** gana (en empate, la
primera en orden de archivo).

- **origen privado/no enrutable** (RFC1918, ULA fc00::/7,
  link-local, loopback): el cliente está DENTRO de la red del
  relay — hereda la posición autodescubierta del propio relay;
- **origen público ausente de la tabla**: mundo (sin división geo
  — nunca una respuesta incorrecta).

**La estrictitud** se conserva para cualquier futura superficie de
tabla: una línea inválida es un error de arranque tipado que lleva
el número de línea infractor; nunca una línea ignorada en
silencio, nunca una tabla parcial.

#### Precedencias de opciones

- **Localizador**: la tabla embebida IMPONE la geolocalización —
  sin sobreescritura por parte del operador. El relay descubre su
  posición desde su dirección externa (STUN/AutoNAT/UPnP resueltos
  por la tabla); los clientes LAN heredan esa posición; los
  clientes de Internet se resuelven por su IP de origen; lo
  desconocido cae al mundo.

Ejemplo (el enrutado geo está activo SIN ningún flag geo):

```bash
scone relay --dns 127.0.0.1:5353
```


## DoH / DoT (transportes cifrados) — vía frontend externo

El servidor DNS de Scone habla **UDP y TCP planos** (RFC 1035 /
7766) — ese es el alcance nativo, completo y probado. Los
transportes cifrados hacia el *cliente final* (DoH —
DNS-over-HTTPS, RFC 8484; DoT — DNS-over-TLS, RFC 7858) están
**delegados a un frontend estándar**: Scone sirve los datos, el
frontend sirve el transporte.

Por qué este diseño: un servidor DoH nativo exigiría una pila
HTTPS completa dentro del nodo (certificados, renovación, HTTP/2)
— una gran superficie operativa para cero beneficio antes de que
haya usuarios finales en el lado del navegador. Los frontends
existentes lo hacen mejor y desde hace más tiempo; el nodo se
mantiene mínimo.

### Ejemplo: unbound haciendo DoH delante de un relay Scone

`/etc/unbound/unbound.conf`:

```
server:
    interface: 127.0.0.1
    port: 5354
    do-ip4: yes
    do-udp: no
    do-tcp: yes

    # DoH (RFC 8484) en 443
    https-port: 443
    tls-service-key: "/etc/unbound/scone.key"
    tls-service-pem: "/etc/unbound/scone.pem"

forward-zone:
    name: "."
    forward-addr: 127.0.0.1@5353   # el relay Scone (--dns 127.0.0.1:5353)
```

El certificado (`scone.pem`/`scone.key`) se obtiene como en
cualquier servicio TLS — p. ej. certbot/acme.sh sobre el dominio
del resolvedor. Los clientes DoH apuntan entonces a
`https://dns.example.tld/dns-query`.

### Ejemplo: knot-resolver haciendo DoT

`/etc/knot-resolver/kresd.conf`:

```
net.listen('127.0.0.1', 5353, { kind = 'dns' })
net.listen('0.0.0.0', 853, { kind = 'tls' })
tls.cert_file = '/etc/knot-resolver/scone.pem'
tls.key_file  = '/etc/knot-resolver/scone.key'
```

### Lo que añade el frontend

- **Cifrado del canal cliente→frontend** (el segmento hostil: la
  red del usuario); el segmento frontend→relay queda local (misma
  máquina o red de confianza).
- **Una caché extra en el lado del frontend** (caché de
  unbound/knot también — inofensiva, los TTL de Scone siguen siendo
  la cota superior).
- **Un endpoint estable** (IPv4/IPv6, dominio) mientras los relays
  se mueven.

### Lo que el frontend NO hace

- No *verifica* nada: cada respuesta servida por el relay ya está
  verificada contra la cadena antes de ser cacheada (ver «Ruta de
  resolución»). El frontend reenvía respuestas
  ya-autoritativas.
- No sustituye la descentralización: un frontend DoH es un
  servicio *operado* — elección del operador del resolvedor, no
  del protocolo. Cada relay Scone sigue siendo utilizable de forma
  independiente sobre UDP/TCP nativos.

### Implementación nativa futura (documentada, sin compromiso)

Un transporte DoH nativo (`--doh ADDR`) sigue siendo concebible:
siendo el formato wire idéntico (RFC 8484 reutiliza los mensajes
DNS binarios), sería un decodificador POST/GET + HTTP/2 tras un
proxy TLS (caddy/nginx gestionando los certificados), quedando el
nodo mismo sin TLS. Sin trabajo en curso — la demanda llegará con
los usuarios finales.
