# Relay de red (M4) — arquitectura, flujos y control CLI

> Especificación normativa del relay de red de Scone
> (`scone-network`). Cualquier divergencia doc/código = bug
> (corregir la fuente de verdad).

## Visión general

El **relay** es el nodo P2P completo de Scone: mantiene la
blockchain canónica (RAM + `NodeStore` redb), reenvía transacciones
y bloques, produce bloques (modo devnet), sirve el protocolo de
sync, publica y resuelve `SignedDnsRecord`s en un DHT Kademlia, y
expone el **protocolo p2p** (`/scone/reqres/1`) que el CLI consulta
directamente (unificación RPC→p2p: el canal JSON TCP desapareció).

```text
                    ┌────────────────────────────────────────┐
                    │              scone relay               │
  P2P (libp2p)      │                                        │
  QUIC (udp,        │  Swarm libp2p                          │
  puerto efímero    │  ├─ identify  (descubrimiento de dir.) │
  por defecto) ◄────┤  ├─ ping      (vida)                   │
                    │  ├─ kad       (DHT de registros)       │
                    │  └─ reqres    (mensajes scone-prot.)   │
                    │                                        │
                    │  Núcleo del relay (tarea única)        │
                    │  ├─ Blockchain (validación completa)   │
                    │  ├─ Mempool (acotado, dedup por TxId)  │
                    │  ├─ BlockBuilder (producción devnet)   │
                    │  ├─ Parking de forks (acotado, 2048)   │
                    │  └─ Driver de sync (puesta al día)     │
                    │                                        │
  CLI ◄─────────────┤  reqres p2p (QUIC, mismos mensajes)    │
                    └────────────────┬───────────────────────┘
                                     │
                              NodeStore (redb)
```

`crates/scone-network` es el **único crate async** del workspace:
todo lo que está por debajo (crypto → core → protocol → blockchain
→ storage) sigue siendo síncrono y puro.

## Admisión de conexiones (P1.5)

Límites duros a nivel de swarm (comportamiento
`connection_limits` de libp2p, la primera entrada de
`SconeBehaviour`):

- `MAX_TOTAL_CONNECTIONS = 128` — tope de conexiones establecidas
  (entrantes + salientes) por relay; el excedente entrante se
  RECHAZA, nunca se pone en cola;
- `MAX_PER_PEER_CONNECTIONS = 4` — un solo par no puede acaparar
  la cuota total abriendo muchas conexiones;
- `MAX_PENDING_INCOMING = 32` — guarda slowloris: las
  semiconexiones no autenticadas jamás esperan indefinidamente.

Estos límites completan los límites por mensaje existentes
(`MAX_MESSAGE_LEN` 1 MiB) y por presupuesto (sync `max ≤ 128`
bloques, DNS TCP 64 conexiones/512 B/5 s). Un atacante por
inundación de conexiones obtiene rechazos, no crecimiento de
memoria.

## Descubrimiento de pares

Tres mecanismos complementarios, en este orden de prioridad:

1. **Seeds explícitos** (`--bootstrap`, multiaddrs completos) —
   siempre tienen prioridad; proporcionar una lista no vacía
   REEMPLAZA todo lo demás. `--no-builtin-seeds` vacía los seeds
   embebidos para despliegues 100% privados.
2. **Seeds embebidos** (`builtin_seeds(network)` en
   `scone-network/src/config.rs`) — congelados en cada release
   oficial del binario, por red. Un relay sin configurar se une a
   la red solo. (Vacío en testnet mientras la testnet pública no
   esté desplegada.)
3. **Directorio aprendido** (`<data-dir>/peers.json`) — toda
   dirección aprendida vía `identify` se retiene (última
   visualización; tope de **2000 entradas** — direcciones, no
   conexiones, ~400 KiB de RAM en el máximo — y un **corte por
   antigüedad de 90 días**: una entrada vista por última vez hace
   más de 90 días se cae en escritura y se ignora en carga,
   manteniendo los 2000 slots llenos de pares plausibles en lugar
   de fósiles pre-churn cuya única contribución sería un timeout
   de marcado al arrancar) y se guarda ~1×/min en el tick de
   producción. Al arrancar sin seed ni bootstrap, el directorio se
   recompone: el relay encuentra a sus pares habituales sin
   configuración. Es una caché: archivo ausente/corrupto =
   directorio vacío, nunca un error.
4. **mDNS LAN** (`--mdns`) — descubrimiento sin configuración en el
   enlace local: los relays vecinos del mismo segmento de red se
   encuentran SIN ningún seed (la ruta devnet sin configuración).
   **Por defecto: ACTIVADO en devnet, APAGADO en las demás** (una
   red local reiniciable es por diseño una red de descubrimiento;
   un despliegue público no multidifunde nada). Comportamiento:
   bind del socket de multidifusión; un fallo de bind (sin
   multidifusión en la interfaz, entorno aislado/sandbox)
   **desactiva** mDNS con una advertencia — nunca fatal (el
   descubrimiento es una comodidad, nunca un salvavidas). Cada
   vecino `Discovered` alimenta la tabla Kademlia
   (`kad.add_address`), exactamente igual que un bootstrap
   explícito menos la configuración; las entradas `Expired` las
   sigue libp2p mismo (la conexión se cierra por su propio
   timeout). Los miembros `mdns`/`upnp` del behaviour son
   `Toggle`s (activados/desactivados en construcción,
   `build_behaviour(key, enable_mdns, enable_upnp)`).
5. **UPnP** (`--upnp`, opt-in) — mapeo del puerto de escucha QUIC
   por el router doméstico: el nodo se vuelve alcanzable desde
   fuera SIN circuit relay. En `NewExternalAddr`, la dirección
   externa se registra como dirección externa real del swarm
   (`add_external_address` — vista por identify/autonat,
   candidata a relay de transporte). Sin pasarela IGD
   (`GatewayNotFound`) o con pasarela no enrutables
   (`NonRoutableGateway`): un simple log info, el mapeo se omite —
   casos normales en una LAN doméstica. Un mapeo caducado
   (renovación fallida) es una advertencia.

mDNS sigue a la red (un `--mdns`/`SCONE_MDNS` explícito siempre
gana): **ACTIVADO por defecto en devnet**, APAGADO en
testnet/mainnet; UPnP queda **desactivado por defecto**
(`enable_mdns`/`enable_upnp` = `false` en `config.rs`, el relay
activa mDNS para devnet): un despliegue público no multidifunde
nada ni habla con ningún router.

### Optimización de topología local (siempre activa)

El relay optimiza activamente su vecindario con el motor puro
`scone-topology` (ver la documentación del crate). **Siempre
activado**; el único ajuste expuesto es un **perfil**
(`--topology-profile`, env `SCONE_TOPOLOGY_PROFILE`, por defecto
`default`):

| Perfil | mín / objetivo / máx | eval. | para quién |
|---|---|---|---|
| `tiny` | 2 / 4 / 6 | 120 s | Raspberry Pi, caja doméstica, VPS 1-vCPU |
| `small` | 4 / 8 / 12 | 90 s | servidor pequeño, NAS doméstico |
| `default` | 8 / 16 / 32 | 60 s | servidor estándar (por defecto) |
| `high` | 16 / 32 / 64 | 45 s | nodo soberano, backbone |
| `very-high` | 32 / 64 / 128 | 30 s | hubs fundadores |
| `devtest` | 2 / 4 / 6 | 5 s | **SOLO LAB/TEST** (inestable por diseño) |

```
scone relay --topology-profile tiny
```

Los parámetros protectores (histéresis 0,15, cooldown 600 s,
gracia 120 s, medición 15 s) son idénticos en todas partes:
protegen la estabilidad y no tienen por qué escalar con el tamaño
del nodo. Un nombre desconocido es un error tipado de arranque.

Una configuración incoherente (`min > target > max`, umbral fuera
de `[0,1]` — `NaN` incluido —, intervalo nulo) es un **error tipado
de arranque**, nunca un comportamiento silencioso.

**Vista (`TopologyView`)** — alimentada por eventos del swarm:

- `ping` exitoso → observación de RTT (ventana de mediana de 8
  muestras);
- `ping` fallido → penalización de fiabilidad;
- `identify` recibido → direcciones (≤ 4/par) + protocolos
  anunciados (máscara kad/reqres/identify/relay);
- conexión establecida → estado conectado, dirección
  (entrante/saliente), edad, circuit-relay detectado;
- conexión cerrada → penalización + contador de churn SOLO si no
  fue solicitada (un desalojo voluntario no cuenta);
- fallo de marcado saliente → penalización de fiabilidad.

La vista está acotada (500 pares; más allá, solo se memorizan los
pares conectados).

**`kad_required` desde los kbuckets** — antes de cada evaluación,
el relay recorre la tabla de enrutado Kademlia: un par está
protegido si está `Connected` en un bucket, o `Disconnected` en un
bucket SATURADO (k = 20). Un par protegido NUNCA se elige para
desalojo.

**Bucle de optimización** — al intervalo de evaluación del perfil
(60 s bajo `default`; ver la tabla arriba):

1. recomputar `kad_required` (kbuckets);
2. evaluación del motor (histéresis: período de gracia de 120 s,
   mejora mínima de puntuación de +15%, cooldown de desalojo
   600 s);
3. decisión aplicada: `AddCandidate` → marcar al candidato;
   `Replace` → cerrar el par desalojado (UNA vez — progresivo, sin
   buclear) luego marcar al reemplazo;
4. una instantánea de **métricas** se registra (nivel INFO):
   recuentos conocidos/conectados/entrantes/salientes/circuitos/
   protegidos, RTT medio/mediano, buckets de IP distintas,
   concentración máxima, churn (reemplazos/desconexiones).

La puntuación del vecindario NUNCA es una medida de confianza (ver
la documentación de `scone-topology`): ninguna decisión de
consenso deriva de ella.

### Alcanzabilidad (AutoNAT) y relay de transporte (circuit-relay v2)

Cada relay ejecuta AutoNAT (cliente + servidor): el nodo pide a sus
pares que intenten un marcado DE VUELTA hacia él; el veredicto
(`reachable`: sí = alcanzable desde fuera, no = tras NAT,
desconocido = aún sin probar) se refresca cada 5 min y se expone en
`NodeStatus` (campo `nat_reachable`, varint wire 0/1/2) — el CLI
lo muestra (`scone status`). Las direcciones privadas cuentan
(`only_global_ips = false`): una malla LAN prueba su alcanzabilidad
LAN.

Un relay alcanzable activa circuit-relay v2 **hop** (puede reenviar
tráfico para pares NATeados) y **DCUtR** (perforación de agujeros:
dos nodos conectados vía circuito intentan una conexión directa y
liberan el relay). Un relay considerado NATeado pide
automáticamente una reserva al primer par alcanzable conocido y
escucha a través de él (dirección
`/p2p/<relay>/p2p-circuit`).

Regla de producción: un nodo NATeado con relay no debería
convertirse en productor de bloques (latencia/disponibilidad
degradadas) — la regla PoS (asiento = dominio vivo) aún no lo
prohíbe; pendiente de documentar como requisito operativo antes de
la testnet pública.

## Transporte y protocolos libp2p

| Protocolo | Nombre | Rol |
|---|---|---|
| QUIC (quic-v1) | — | Transporte principal (udp) |
| identify | `/scone/id/1` | Descubrimiento de direcciones de pares (llena la tabla Kademlia) |
| ping | estándar libp2p | Vida |
| Kademlia | `/scone/kad/1` | DHT: clave = `DomainId` (32 B), valor = codificación canónica del `SignedDnsRecord` |
| request-response | `/scone/reqres/1` | mensajes de aplicación de scone-protocol |

### Trama de aplicación (`/scone/reqres/1`)

Cada trama es `len: u32 BE || bytes` donde los bytes son la
codificación canónica de un `Message` de `scone-protocol` (el
códec existente — sin segundo formato). La longitud está acotada
por `limits::MAX_MESSAGE_LEN` **antes** de cualquier lectura o
asignación: un par que anuncie una trama mayor se rechaza sin coste
de memoria.

Mensajes intercambiados (uno por round-trip petición/respuesta):

| Mensaje | Dirección | Respuesta |
|---|---|---|
| `Hello { version }` | handshake | `Hello` (versión local) |
| `Ping` / `Pong` | vida | `Pong` |
| `GetBlock { hash }` | consulta | `Block` (ventana RAM) o `Pong` si desconocido |
| `GetBlocks { start, max }` | sync | **un** `Block` (altura `start`), o `Pong` si ninguno |
| `Block (b)` | relay de bloques | `Pong` (ack) |
| `Transaction (tx)` | relay de tx | `Pong` (ack) |
| `GetRecord { domain_id }` | DHT local | `Record` (caché local) o `Pong` |
| `Record (r)` | publicación | `Pong` si aceptado, error si no |

Regla: **exactamente una respuesta por petición**; los mensajes de
acknowledgment son `Pong`. Las respuestas de sync llevan un bloque
a la vez — la puesta al día itera en el lado solicitante (ver
«Sync»). Esto está deliberadamente acotado: nunca más de un bloque
en vuelo por par.

## Comportamiento del relay

### Arranque

0. resolución de red (`--network testnet|mainnet|devnet`, testnet
   por defecto mientras el proyecto está en desarrollo — M8b). El
   data-dir es **por red** (`~/.scone/<network>/`): un almacén no
   vacío que contenga una cadena de otra red se rechaza
   (`Corrupted`);
1. abrir/crear el almacén redb (`<data-dir>/chain.redb`);
2. `load_chain(store, network)`: tip O(1) + estados de dominios
   paginados (ventana RAM: génesis + tip);
3. construcción del swarm (identidad Ed25519 efímera por sesión —
   persistir la identidad de red es una decisión abierta);
4. componer la lista de arranque (ver [Descubrimiento de
   pares](#descubrimiento-de-pares)): `--bootstrap` explícito
   (debe terminar en `/p2p/<peer-id>`) SI NO los seeds embebidos de
   la red SI NO el directorio aprendido (`peers.json`);
5. marcar cada multiaddr y registrarlo en la tabla Kademlia
   (un marcado muerto se ignora — los demás componen la red).

### Recepción de una transacción (P2P o envío CLI `submit tx`)

1. decodificación estricta acotada (`MAX_MESSAGE_LEN`);
2. validación criptográfica completa (`validate_transaction`:
   ligadura owner/clave, `verify_strict` sobre el payload
   canónico);
3. **anti-replay PRIMERO** (barato, independiente del estado,
   definitivo), luego pre-comprobación de estado (espejo barato de
   las reglas de `push_block` — el PoW, costoso, solo se verifica
   en el push): un `RenewDomain` reproducido responde `TxReplay`,
   no un error de PoW mentiroso (su prueba liga la expiración que
   la primera aplicación ya superó). La separación de redes se
   aplica en el paso 2: el payload de la firma inyecta la red del
   RELAY, una tx firmada en otra parte falla con
   `InvalidSignature` (wire 2026-09: sin campo `network`);
   `RegisterDomain` → TLD del nombre registrado (`UnknownTld` en
   caso contrario, D1/M7c) **y abierto** (`TldClosed` en caso
   contrario, M8b) luego nombre libre; `UpdateDomain` → dominio
   conocido, secuencia exacta `current + 1`, propietario;
   `RegisterTld` → TLD libre; `TransferTld`/`RevokeTld`/
   `SetTldOpen` → TLD conocido + firmante propietario;
   `AssignDomain` → propietario del TLD + nombre libre;
   `RenewTld` → TLD privativo conocido + propietario, plazo fijo
   rederivado, pre-comprobación de ventana de gracia;
   `RenewDomain` → dominio conocido + propietario,
   pre-comprobación del PoW de renovación (ligado a la expiración
   actual) a la altura siguiente;
4. inserción en mempool (capacidad fija, dedup por `TxId` —
   reenviar una tx ya en el pool es un no-op);
5. difusión a los pares (salvo el enviador).

### Recepción de un bloque (P2P, sync o autoproducido)

1. `push_block`: validación completa existente (padre = tip
   canónico, altura, raíz Merkle recomputada, cada transacción
   validada luego aplicada — atómica por bloque);
2. si se acepta: el relay lee el resultado de la aplicación
   (`push_block_with_gc` → hash + dominios eliminados por el GC
   determinista) y persiste vía `store_block_with_removals`
   (append atómico de deltas: bloque + tip + estados modificados —
   upserts Y eliminaciones: dominios caducados, TLDs revocados).
   Un reinicio no puede pues resucitar un registro caducado;
3. reconciliación del mempool (las txs incluidas se eliminan);
4. difusión a los pares (salvo el enviador).

Un bloque inválido se rechaza con un error tipado; el relay sigue
corriendo. Sin panic con datos de red. Las aplicaciones pesadas
(Argon2 + commit redb) pasan por `tokio::task::block_in_place`
(driver síncrono, aceptación p2p, producción): el runtime mueve
las tareas listas a otros trabajadores — aplicar un bloque ya no
detiene el servicio DNS ni las métricas.

### Snapshot DNS (publicación)

Tras CADA bloque aplicado (aceptación p2p — reorgs incluidos —,
driver de sync, producción) Y al arrancar (un reinicio no debe
hacer NXDOMAIN a nombres vivos hasta el bloque siguiente), el
relay publica el **snapshot DNS autoritativo** en el feed `watch`
que lee la superficie DNS (especificación completa en `dns.md`):

- extensión del tip (la ruta abrumadora): **delta O(tocados)**,
  copy-on-write — las transacciones del bloque nombran los
  dominios tocados, más las eliminaciones del GC determinista
  (`AppliedBlock.gc_removed_domains`, evaluadas en el push);
- reorg / arranque: **reconstrucción completa** (un paseo paginado
  de `ChainState::dns_facts`).

### Producción de bloques (M5: productor firmado)

Cada `produce_interval` segundos (2 s por defecto): si el mempool
no está vacío, un bloque es ensamblado por `BlockBuilder`
(`timestamp = now`) **firmado por un productor autorizado** (payload
`SCONE-BLOCK-V1` en `header.consensus`, ver
`/docs/en/technical/blockchain.md`) y sigue exactamente la ruta de
aceptación de un bloque recibido. Un relay sin txs no produce
nada.

Clave de producción: la clave anchor (`--anchor-key`) cuando el
relay está armado con una — un anchor es un productor autorizado —,
si no la clave devnet fija, válida mientras el pool de productores
esté vacío (bootstrap abierto: génesis). En cuanto exista un
dominio/TLD vivo, un bloque firmado por la clave devnet se rechaza
(`InvalidProducer`).

### Bucle anchor (finalidad, M5)

Con `--anchor-key` (archivo de claves `.sconekey`, passphrase vía
`--anchor-passphrase-env`, por defecto `SCONE_ANCHOR_PASS` — el
valor por defecto de la biblioteca `scone-network`
`SCONE_ANCHOR_PASSPHRASE` nunca se alcanza a través del binario,
clap siempre lo sobreescribe):

- tras cada bloque aceptado/producido, si este nodo es miembro del
  comité actual **y** el comité alcanza
  `MIN_FINALITY_COMMITTEE_SIZE`, el relay firma el checkpoint
  propuesto vía su `SignerGuard` (`signer-state.dat` en el
  data-dir — una clave = un voto/época, fail-closed a prueba de
  caídas) y lo glosipea;
- las firmas recibidas (`Message::Checkpoint`) se acumulan por
  hash de `CheckpointData` (acotado a 256 entradas); a
  `quorum_for(tamaño del comité)` firmas de miembro sobre los
  MISMOS datos → `accept_checkpoint` + difusión del checkpoint
  finalizado;
- un relay sin clave anchor queda **pasivo**: acumula, finaliza y
  reenvía, pero nunca firma;
- `GetCheckpoints` → se sirve la ventana `checkpoint_window()`;
- cualquier error del bucle se registra y se traga — nunca fatal;
- el estado (`GetStatus`) expone `checkpoints` y `finalized_epoch`.

### Parking de forks y reorgs (corrección minoría-pegajosa)

Un bloque hermano a la misma altura, o un bloque cuyo padre no es
el tip actual (`ParentNotTip`/`UnknownParent`), NUNCA se pierde:
se **aparca** (mapa acotado, `PARKED_CAP = 2048` bloques, ~KiB
cada uno — RAM de peor caso acotada; un parking lleno ignora
silenciosamente a los recién llegados). Los hashes ya ofrecidos a
`accept_block` se deduplican (`seen_blocks`): un bloque perdedor
re-glosipeado no se re-evalúa.

En CADA tick `Produce`, `reconsider_parked`:

1. drenaje lineal de los bloques aparcados cuyo padre se volvió el
   tip;
2. si una rama aparcada es más fuerte que el tip actual (reglas de
   elección de rama SIN CAMBIOS: piso de finalidad, cadena más
   larga, menor hash), adopción — el sufijo desplazado se aparca él
   mismo (AMBOS lados del fork siguen evaluables).

Esto despegoida un nodo que adoptó el lado perdedor de un fork de
igual altura: el bloque ganador, ofrecido una vez y luego
rechazado, se re-evalúa periódicamente en lugar de olvidarse para
siempre (hallazgo chaos-30: dos bloques hermanos a 15 ms).

**Persistencia del reorg**: adoptar otra rama no puede pasar por
el append monótono (`NonMonotonicHeight`). El relay aplica
`rewind_to_height(punto de bifurcación)` luego
`persist_state_snapshot` (reescritura del estado vivo desde la
exportación de la cadena) luego un re-append monótono del sufijo
ganador (ver `storage.md`). Un fallo de persistencia se registra y
se degrada (se cura al arrancar vía replay) — nunca fatal.

## Sync (puesta al día)

En cada conexión a un par, el relay pide
`GetBlocks(start = altura_local + 1)`. Cada `Block` recibido que
completa exactamente la altura esperada dispara la petición de la
altura siguiente, hasta el agotamiento (`Pong` = nada más) o
`MAX_SYNC_ROUNDS` (10.000) lotes — una cota absoluta incluso contra
un par mentiroso. Los bloques recibidos pasan por validación
completa: un par malicioso no puede hacer aceptar nada inválido.

### DHT de registros

- **put** (flujo API/web — el canal hex del CLI fue eliminado):
  decodificación estricta acotada (`MAX_DHT_CACHE_ENTRY`),
  verificación contra el estado on-chain (owner, secuencia,
  `record_hash` recomputado — ver «Verificación de registros»),
  luego `kad.put_record` (almacenamiento local + replicación a los
  pares más cercanos, `Quorum::Majority` del factor de
  replicación) y caché redb persistente.
- **get** (`domain resolve`): `kad.get_record`; el waiter queda
  ligado a su `QueryId` y al `DomainId` pedido — una respuesta solo
  resuelve la consulta coincidente, y la clave del registro debe
  ser exactamente el dominio pedido (comprobado **antes** de la
  verificación on-chain). El resultado se verifica luego contra la
  cadena antes de devolverse. Tope de peticiones simultáneas:
  `MAX_DHT_WAITERS` (256); más allá, error inmediato. Una
  resolución verificada con éxito se escribe en la caché
  persistente (M5): las lecturas locales (`domain_info`) la sirven
  luego sin nueva consulta DHT.
- Kademlia corre en modo servidor, sin TTL de registros (la cadena
  gobierna la validez, no el tiempo).

### Verificación de registros (regla «la cadena es la autoridad»)

Un `SignedDnsRecord` resuelto o recibido solo se acepta/devuelve
si:

1. el dominio está registrado on-chain;
2. `record.owner` = propietario on-chain actual;
3. `record.sequence` = secuencia on-chain actual;
4. `record_hash(record.record)` (BLAKE3 recomputado) = el
   `record_hash` comprometido on-chain.

Si no: rechazo silencioso en el lado P2P, error explícito en el
lado CLI. La firma Ed25519 del registro se **verifica a la
recepción** por el relay (`ResolveLocal`: la clave pública del
propietario se conoce on-chain, la firma debe validar — un registro
firmado por otra clave se rechaza, `record rejected`); también se
transporta para los verificadores futuros — el compromiso on-chain
sigue siendo el control de integridad primario.

## Métricas Prometheus

Un endpoint HTTP (formato texto Prometheus 0.0.4, exposición
hecha a mano — sin dependencia nueva) servido por una tarea
dedicada:

- **Público por defecto** en `127.0.0.1:9101` (el CLI aplica ese
  valor; el de la biblioteca es `None` — varios relays en un
  proceso nunca colisionan).
- `--metrics-addr ADDR|off`: otra dirección, o `off` = endpoint
  desactivado.
- `--metrics-token TOKEN`: pasa a modo **privado** — `Authorization:
  Bearer ***` requerido (comparación en tiempo constante, `401` en
  caso contrario; el token nunca aparece en logs ni en la
  exposición).
- Un fallo de bind **desactiva el endpoint** (advertencia no fatal)
  — nunca el nodo.

El bind efectivo se anuncia en stdout (contrato legible por
máquina, como `p2p:`): una línea `metrics: <addr>` tan pronto como
escucha.

**19 contadores** (`counter`, monótonos — rutas calientes:
`fetch_add(Relaxed)` sobre un `Arc<Metrics>` compartido, nunca una
entrada de consenso):

| Grupo | Series |
|---|---|
| Cadena | `scone_blocks_applied_total`, `scone_blocks_rejected_total`, `scone_checkpoints_signed_total`, `scone_checkpoint_sigs_received_total` |
| Mempool | `scone_txs_admitted_total`, `scone_txs_rejected_total`, `scone_txs_evicted_grace_total`, `scone_txs_included_total` |
| DNS | `scone_dns_queries_total`, `scone_dns_cache_hits_total`, `scone_dns_cache_misses_total`, `scone_dns_local_chain_total`, `scone_dns_dht_answers_total`, `scone_dns_rejected_unverified_total` |
| DHT | `scone_dht_pulls_issued_total`, `scone_dht_negative_hits_total`, `scone_dht_negative_expired_total`, `scone_dht_negative_capped_total`, `scone_dht_cache_evictions_total` |
| Red de pares | `scone_peer_connections_established_total`, `scone_peer_connections_closed_total`, `scone_peer_connections_incoming_total`, `scone_peer_dial_failures_total`, `scone_peer_blocks_received_total`, `scone_peer_transactions_received_total` |

**6 gauges** (`scone_height`, `scone_mempool`, `scone_peers`,
`scone_domains`, `scone_finalized_epoch`, `scone_addr_book_entries`)
— los cinco primeros refrescados cada 15 s vía el canal
`RelayCommand` (misma ruta que el `status` del CLI — fuente única
de verdad); `scone_addr_book_entries` lo mantiene directamente el
registro (actualizado en cada inserción en el libro de direcciones
— fuente de verdad).

Test E2E: `metrics_endpoint_public_and_private` (200 + exposición,
401 sin token/con token erróneo, 200 con Bearer).

## Control CLI (unificación RPC→p2p)

El servidor RPC JSON TCP ha sido **eliminado**: el CLI (y el
puente axum del registrar) hablan el protocolo p2p
**directamente** — mismo wire (`/scone/reqres/1`), mismos
handlers en el lado del relay, cero canal paralelo. Cada comando
abre una conexión QUIC al multiaddr del nodo (`--peer`),
intercambia UN `Message`, cierra.

Mensajes de control servidos por el relay:

- `GetStatus` → `NodeStatus { network, nat_reachable, tip,
  height, domain_count, mempool, peers, finalized_epoch,
  checkpoints, fresh_operators }` (mostrado por `scone status` —
  `fresh_operators` = operadores con heartbeat fresco, RFC
  proof-of-service);
- `LookupName { name }` → `NameState` (estado on-chain, como
  `lookup`);
- `GetOwnedNames { owner }` → `OwnedNames` (dominios + TLDs vivos
  de un `OwnerId`);
- `Transaction (tx)` → envío con gossip libre (el relay
  re-difunde a los pares);
- `GetRecord { domain_id }` / `Record (r)` — resolución y
  publicación DHT (como `scone domain resolve` en el lado CLI);
- rechazo de aplicación → `Message::Error(msg)` (llega al cliente
  como mensaje legible, no como EOF).

El cliente levanta un swarm libp2p mínimo (request-response +
identify), timeout de llamada de 30 s (20 s de round trip).
`--peer` por defecto:
`/ip4/127.0.0.1/udp/4001/quic-v1/p2p/12D3KooW…` (nodo local de
dev; el multiaddr completo con el peer id real lo anuncia el relay
en stdout vía la línea `p2p:`).

El canal interno `RelayCommand` sobrevive para la superficie DNS
embebida (`ResolveLocal`, un round trip en-proceso, nunca un
waiter DHT — ver `dns.md`).

## CLI

```
scone relay [--network testnet|mainnet|devnet] [--data-dir DIR] [--listen MULTIADDR] [--bootstrap MULTIADDR]... [--no-builtin-seeds]
            [--mdns] [--upnp] [--rpc ADDR (obsoleto, ignorado)]
            [--anchor-key PATH] [--no-anchor-auto] [--anchor-passphrase-env VAR] [--genesis FILE]
            [--geo-config FILE] [--geo-ip-table PATH] [--geo-country U16] [--geo-continent U16] [--geo-asn U32] [--geo-region U32]
            [--health-check] [--no-health-check] [--health-ttl-secs U64] [--health-port U16]
            [--topology-profile tiny|small|default|high|very-high|devtest]
            [--dns ADDR] [--dns-upstream ADDR]... [--dns-cache-limit U] [--dns-cache-max-mem SIZE] [--dns-max-inflight U]
            [--metrics-addr ADDR|off] [--metrics-token TOKEN]
scone status [--peer MULTIADDR]
scone domains [--identity ID | --pubkey HEX] [--peer MULTIADDR]
scone domain register NAME [--identity ID] [--dir DIR] [--passphrase-env VAR] [--peer MULTIADDR]
scone domain set NAME TYPE VALUE [TTL] [--identity ID] [--peer MULTIADDR]   # registro único exprés
scone domain update NAME --file FILE [--identity ID] [--peer MULTIADDR]
scone domain get NAME [--peer MULTIADDR]
scone domain renew NAME [--identity ID] [--peer MULTIADDR]
scone domain resolve NAME [--peer MULTIADDR]

# --peer apunta por defecto al relay local (el env SCONE_PEER lo sobreescribe);
# --identity apunta por defecto a la identidad única del keystore cuando hay
# exactamente una. Eliminados (los stubs sugieren reemplazos): show,
# submit, lookup, owned.
```

Cada comando de control habla QUIC directamente al multiaddr del
relay (`--peer`, nodo local por defecto — unificación RPC→p2p, sin
canal JSON TCP). Relay ausente → mensaje claro «p2p node
unreachable — is a relay running?», salida no nula. El flag
`--rpc` se conserva pero es **obsoleto e ignorado** (el servidor
RPC fue eliminado; solo por compatibilidad de scripts).

### Configuración por entorno (`.env`)

Al arrancar, `scone` carga (vía `dotenvy`, antes del análisis de
clap) un archivo `.env` del directorio actual: archivo ausente =
no-op silencioso, nunca un error. Los valores cargados se
convierten en la **línea base de entorno** de las opciones que
declaran una variable (`env = "…"` en `cli.rs`); **un flag CLI
explícito siempre gana** sobre `.env`, y `.env` gana sobre los
valores por defecto integrados:

```text
flag CLI explícito  >  variable .env  >  valor por defecto integrado
```

El archivo `.env.example` en la raíz del repositorio documenta
cada variable (copiar a `.env` y ajustar; `.env` está
**gitignored** — nunca hacer commit de un `.env` real). Variables:

| Variable | Flag espejo | Semántica |
|---|---|---|
| `SCONE_NETWORK` | `--network` | `testnet` (por defecto), `mainnet`, `devnet` |
| `SCONE_DATA_DIR` | `--data-dir` | directorio de datos (por defecto `~/.scone/<network>/`) |
| `SCONE_LISTEN` | `--listen` | multiaddr de escucha P2P (por defecto: todas las interfaces, puerto efímero) |
| `SCONE_BOOTSTRAP` | `--bootstrap` | seeds de bootstrap, **repetible** (una línea por valor — dotenvy acumula); reemplaza los seeds integrados |
| `SCONE_GENESIS` | `--genesis` | archivo de génesis (nodos fundadores) |
| `SCONE_MDNS` | `--mdns` | descubrimiento mDNS LAN (`true` = activado, `false` = apagado; por defecto: ACTIVADO en devnet, APAGADO en las demás) |
| `SCONE_UPNP` | `--upnp` | mapeo de puertos UPnP (`true` = activado) |
| `SCONE_DNS_ADDR` | `--dns` | dirección de bind del servidor DNS UDP |
| `SCONE_DNS_UPSTREAM` | `--dns-upstream` | fallbacks recursivos, **repetible** |
| `SCONE_DNS_CACHE_LIMIT` | `--dns-cache-limit` | tope de entradas de la caché DNS |
| `SCONE_DNS_CACHE_MAX_MEM` | `--dns-cache-max-mem` | tope de memoria de la caché DNS (p. ej. `64MB`) |
| `SCONE_DNS_MAX_INFLIGHT` | `--dns-max-inflight` | tope de consultas DNS concurrentes |
| `SCONE_METRICS_ADDR` | `--metrics-addr` | endpoint de métricas (`ADDR` u `off`) |
| `SCONE_METRICS_TOKEN` | `--metrics-token` | token bearer del endpoint de métricas |
| `SCONE_TOPOLOGY_PROFILE` | `--topology-profile` | perfil de topología (`tiny`/`small`/`default`/`high`/`very-high`/`devtest`; por defecto `default`) |
| `SCONE_ANCHOR_KEY` | `--anchor-key` | archivo de claves anchor (checkpoint + producción) |

`SCONE_ANCHOR_PASS` NO es la opción de una variable: es el NOMBRE
POR DEFECTO de la **variable que contiene la passphrase del
anchor** (`--anchor-passphrase-env`, por defecto
`SCONE_ANCHOR_PASS` a nivel CLI) — el relay lee la passphrase de
la variable que este flag nombra.

**Anchor auto (ON por defecto)**: sin `--anchor-key`, un archivo
de claves anchor se genera/persiste automáticamente en el
directorio de datos (`anchor.sconekey`, passphrase
`SCONE_ANCHOR_PASS` si está definida, si no generada al azar y
escrita en `anchor.pass`) — el relay se vuelve productor en cuanto
su clave posea dominios/TLDs vivos (la elegibilidad PoS nunca es
gratis). `--no-anchor-auto` mantiene el relay estrictamente
pasivo. La variable de entorno por defecto de
`--anchor-passphrase-env` es `SCONE_ANCHOR_PASS`.

`--network`: el valor por defecto sigue al BUILD (mainnet en
binarios con mainnet habilitada, testnet en caso contrario);
`devnet` es la red local reiniciable (génesis `scone-devnet`
propio, PoW casi nula de 0..=2 bits — borrar `~/.scone/devnet/` y
relanzar reinicia la cadena). Fija el génesis, las dificultades
PoW y el data-dir por defecto (`~/.scone/<network>/`); una tx de
otra red se rechaza de forma tipada (`WrongNetwork`).

La superficie hex offline (los subcomandos `tx` y `submit`)
desapareció (remove-tx-surface-v2). El CLI cubre las rutas de
usuario (`domain register`, `domain renew`, `domains`) Y la
familia M8b como comandos de primera clase: `tld open`, `tld
transfer`, `tld revoke`, `domain assign`, `domain transfer`. Las
reclamaciones `register` minan su PoW a la dificultad de la red
(milisegundos en devnet).

Las opciones `--geo-*`/`--health-*` configuran el GeoDNS y la
comprobación de salud **del resolvedor DNS embebido**: semántica
completa, orden de precedencia y tabla GeoIP local documentados en
[`dns.md`](dns.md#geodns-geographic-routing) — el enrutado
geográfico está **siempre activo** (tabla GeoIP embebida, sin
opción requerida); `--health-check` está apagado por defecto, ruta
clásica sin cambios.

### `domain register|update` (M5) — transacciones firmadas en un comando

Estos comandos encadenan construir → firmar (keystore) → enviar →
(confirmación) en una sola invocación.

- **register**: construye un `RegisterDomain` firmado (timestamp =
  now, proof vacío en devnet), lo envía, espera la confirmación
  on-chain (secuencia ≥ 0, timeout 30 s). Un nombre ya registrado
  se rechaza en el lado cliente Y en el lado relay.
- **update**: el archivo de registros es la única fuente de
  verdad. La secuencia (on-chain + 1) y el `record_hash` (BLAKE3
  canónico de los registros del archivo) se **derivan**, nunca se
  introducen: firmante y verificadores no pueden divergir. Tras la
  confirmación del UpdateDomain, el `SignedDnsRecord` (registros
  del archivo + owner + firma Ed25519 sobre la codificación
  canónica) se publica al DHT vía `put_record` (que reverifica
  contra la cadena). Un dominio sin registrar, una identidad no
  propietaria o un error de archivo fallan limpiamente, sin tocar
  la cadena.

#### Archivo de registros

Texto, un registro por línea, `#` = comentario, líneas en blanco
ignoradas; la codificación canónica ordenada hace que el orden del
archivo no tenga efecto sobre el hash comprometido. El archivo es
el conjunto COMPLETO (un update reemplaza, no fusiona). Tipos:

```text
A 192.0.2.1
AAAA 2001:db8::1
CNAME www.example.uip
NS ns1.example.uip
MX 10 mail.example.uip
TXT texto libre (espacios normalizados)
```

Errores tipados: tipo desconocido, IP/nombre inválido (con número
de línea), conjunto vacío, duplicado, > `MAX_RECORDS_PER_SET`.

## `--genesis` (fundadores)

`scone relay --genesis genesis.bin` — **solo nodos fundadores**
(campaña kickstarter; el archivo lo produce el binario
`scone-genesis`). Reglas normativas:

1. El archivo se rechaza **antes de leerlo** si supera 1 MiB,
   luego se decodifica estrictamente (`decode_complete`: sin bytes
   sobrantes). El hash del bloque 0 se **recomputa**, nunca se lee
   del archivo.
2. Validación semántica: altura 0, `consensus` empezando por el id
   de red de la red pedida (`--network`), sin transacciones,
   manifiesto de asignación canónico. Cualquier violación = error
   tipado al arrancar (nunca un panic, el daemon no arranca).
3. Data-dir **virgen**: la cadena nace sobre el génesis asignado —
   las asignaciones se aplican en RAM **antes de cualquier
   escritura en disco**, luego se persisten (estados sembrados +
   bloque génesis bajo una clave meta). El primer bloque producido
   solo puede pues extender el génesis asignado.
4. Data-dir **ya productivo**: el génesis de nacimiento del
   almacén es el `prev_hash` del bloque 1 (autocertificado). Hash
   de archivo igual = OK silencioso (la opción es **redundante
   tras el primer arranque**); distinto = rechazo tipado — un nodo
   se une a una red vía sync, nunca vía `--genesis`.
5. Reinicio **sin** la opción: un data-dir nacido de fundador
   recupera su génesis asignado vía la clave meta (re-ligadura RAM
   en cada arranque).

El fijado del hash de génesis constante (anti divergencia de red)
es una línea de trabajo aparte.

### Manifiesto del binario `scone-genesis` (kickstarter)

El binario `scone-genesis` (crate `scone-genesis`) prepara el
manifiesto `genesis-manifest.json` (formato `ALLOC-V1`) y luego
produce `genesis.bin`. Reglas normativas:

- `scone-genesis new --network <red>` crea un manifiesto vacío;
  `add`/`remove`/`list` gestionan las asignaciones, `build`
  produce `genesis.bin` (+ hash), `verify` lo revalida.
- `add domain <nombre>` exige que el **TLD del dominio ya esté
  asignado en el manifiesto** (rechazo inmediato en caso contrario
  — la cadena lo habría rechazado en la aplicación del génesis);
  su `valid_until` es **automático**: `born_at + 1 año` (el
  default_validity del manifiesto), nunca introducido a mano.
- `add tld <nombre>`: **abierto por defecto** (de nadie,
  autoservicio, piso de 1 año, owner = cero en el wire — kind
  `0x03`); `--private` = privativo (owner, plazo de génesis
  `born_at + TLD_TERM_SECS` — 3 años, ciclo de vida estándar:
  `RenewTld`, gracia de 60 días).
- Orden canónico por nombre (contenidos iguales → mismo bloque →
  mismo hash, sea cual sea el orden de inserción); duplicados
  rechazados.

## Límites y garantías de memoria

| Recurso | Cota |
|---|---|
| Trama P2P | `MAX_MESSAGE_LEN` (1 MiB) antes del análisis |
| Mempool | 4096 tx (rechazo tipado más allá); `MAX_PENDING_PER_DOMAIN = 4` tx pendientes por dominio |
| Gracia del mempool | `MAX_GRACE_ATTEMPTS = 10` reinserciones de una tx expulsada por un motivo no definitivo (≈ 7 min a 40 s/ciclo), luego desalojo |
| Parking de forks | `PARKED_CAP = 2048` bloques competidores |
| Checkpoints acumulados | `MAX_PENDING_CHECKPOINTS = 256` (desalojo: menor hash de datos) |
| Waiters DHT simultáneos | `MAX_DHT_WAITERS` (256); búsqueda DHT acotada a 30 s (`DHT_LOOKUP_TIMEOUT`) |
| Bloques por respuesta de sync | 1 por round-trip; ≤ 10.000 lotes por par (`MAX_SYNC_ROUNDS`); cota de protocolo `MAX_BLOCKS_PER_REQUEST = 128` en la petición |
| Registro DHT | `MAX_DHT_CACHE_ENTRY` (64 KiB) |
| Caché DHT persistente (disco) | doble cota LRU: `DHT_CACHE_MAX_ENTRIES` (100.000 entradas) Y `DHT_CACHE_MAX_BYTES` (2 GiB) — la que se alcance primero desaloja por orden de clave; desalojo GC en la MISMA transacción que el bloque (un dominio matado por el GC de la cadena nunca deja una entrada huérfana) |
| Caché negativa DHT | `DHT_NEGATIVE_TTL_SECS` (30 s), tope FIFO `DHT_NEGATIVE_CAP` (4.096) |
| Cadena en RAM | génesis + ventana de tip (el historial se sirve desde el almacén) |

Regla anti-bucle (H2): una transacción solo se re-difunde si fue
**realmente insertada** en el mempool (primera vista); un
duplicado (ya en el pool o ya en la cadena) nunca se re-reenvía.
Un bloque ya canónico se detecta e ignora antes de `push_block`
(sin re-almacenar, sin re-difundir).

Decisiones abiertas: identidad de red persistente del relay, gossip
de difusión (actualmente: request-response unicast a cada par
conocido). Resueltas desde la redacción inicial: elección de rama
real (parking + reconsider_parked, ver arriba), consenso PoW
(productor firmado + comité anchor), resolvedor DNS (la superficie
`--dns`, ver dns.md), canal de control (unificación RPC→p2p: sin
JSON TCP).

## Apagado controlado (SIGTERM / SIGINT)

El daemon `scone relay` (primer plano) captura SIGTERM y SIGINT
vía `tokio::signal`: con la señal, el future `Relay::run` se
cancela (`tokio::select!` en el binario), lo que suelta el Swarm
libp2p y cierra los sockets QUIC. El runtime de tokio se detiene
luego con un apagado acotado (≤ 2 s) que suelta las tareas DNS
desprendidas (UDP/TCP) y cierra sus sockets — el objetivo es que
un `kill` no deje socket UDP huérfano (caso observado en la
testnet 60). Un apagado por señal registra `relay shutting down
(signal)` (INFO, stderr) y sale con código de salida **0**: un
apagado solicitado no es un fallo. Si la instalación del handler
falló, se aplica el comportamiento por defecto de la señal
(degradación, nunca un panic).


## Transporte Tor (cuestión abierta)

QUIC/UDP no atraviesa proxies SOCKS planos, así que los relays no
pueden hoy correr trivialmente sobre Tor. Un transporte Tor nativo
de libp2p (relay con dirección onion) es la vía seguida — ver la
guía de resistencia a la censura del sitio web para el estado
actual y el puente CNAME `.tor` (nombres legibles para servicios
ocultos, soportado hoy).
