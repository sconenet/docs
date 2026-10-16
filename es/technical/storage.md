# Almacenamiento — persistencia local del nodo (redb)

> Documento normativo. Cualquier divergencia entre este documento y
> `crates/scone-storage/src/` es un bug (identificar la fuente de
> verdad y corregirla). Referencia de implementación: crate
> `scone-storage` (formato v1), backend `redb` 4.2.

## Propósito

La persistencia local embebida del nodo: bloques canónicos, estados
de dominios, caché DHT, metadatos. El almacén es una
**persistencia**, nunca una autoridad: el estado RAM
([`ChainState`], la fuente de consenso) se reconstruye al arrancar
desde el almacén, y toda escritura es un **delta** (dominios
modificados por un bloque), nunca una copia completa del estado.

Restricción transversal (orientada al usuario, en cada hito):
**huella de memoria mínima**. Los grandes volúmenes (bloques,
estados de dominios, caché DHT) viven en redb y se acceden vía
cursores/rangos — nunca se cargan completos en RAM, nunca se
duplican inútilmente.

## Frontera: el trait `NodeStore`

`crates/scone-storage/src/lib.rs` define la única cara de
almacenamiento visible para el resto del código (blockchain, futuro
relay M4):

- `append_block(height, hash, bytes)` /
  `append_block_with_state(..., deltas)` — append atómico (ver
  abajo);
- `block_at_height(h)` / `block_by_hash(hash)` — lecturas
  puntuales;
- `tip() -> (height, hash)` — O(1), leído de `meta`;
- `put_domain_state` / `domain_state` / `domain_count` — contador
  **mantenido** (`meta["domain_count"]`), nunca un `COUNT(*)`;
- `iterate_domains(after, max) -> DomainPage` — paginación por
  cursor (lotes de 100 por defecto), orden ascendente de bytes del
  `DomainId`;
- `put_tld_state` / `tld_state` / `tld_count` / `iterate_tlds` —
  mismos contratos para el registro de TLDs (`tld_count` mantenido,
  cursor acotado) (M7d);
- `put_dht_cache` / `dht_cache` — bytes opacos de
  `SignedDnsRecord`;
- `resolve_name(canonical)` / `resolve_tld(tld)` — índice
  persistente Nombre→Id (P0.2), búsqueda puntual O(1), sin iteración
  sobre dominios (por defecto `None` = backend sin índice, el
  llamante cae a `iterate_domains` o al estado de la cadena);
- `tld_floors_all` / `snapshot_tld_floors_all` — lectura de los
  pisos de vida de los TLDs abiertos (opción del propietario:
  tablas dedicadas, `tld_floors_v1` viva y `snapshot_v1_floors`
  congelada);
- `rewind_to_height(h)` — reorg: elimina los bloques por encima de
  `h` y devuelve `tip`/`tip_height` a `h` en UNA transacción
  (preliminar de `persist_state_snapshot`, ver abajo);
- `meta_get` / `meta_set`;
- `snapshot_meta` / `snapshot_domains` / `snapshot_tlds` — lectura
  del snapshot de estado de arranque (M6b; valores por defecto
  vacíos = backend sin snapshot, `load_chain` cae entonces a la
  carga completa).

Reglas:

- **Sin tipos redb en la API**: solo bytes (`&[u8]`, `[u8; 32]`) y
  tipos core. El backend es reemplazable.
- **Sin async, sin mutabilidad interior**: las escrituras toman
  `&mut self` (redb serializa escritores), las lecturas `&self`
  (una transacción de lectura corta por llamada).
- La API nunca revalida bloques: el hash pasado a `append_block*`
  es el hash **recomputado** suministrado por el llamante
  (`Blockchain::push_block`).

## Esquema de tablas redb

| Tabla | Clave (orden) | Valor |
|---|---|---|
| `blocks_by_height` | altura `u64` (b-tree ordenado) | `hash(32) ‖ height(8 BE) ‖ bloque canónico` |
| `blocks_by_hash` | hash del bloque (32 B) | codificación canónica del bloque |
| `domains` | `DomainId` (32 B, orden de bytes) | `DomainStateBytes` (abajo) |
| `tlds` | `TldId` (32 B, orden de bytes) | `TldStateBytes` (abajo, M7d) |
| `dht_cache` | `DomainId` (32 B) | bytes de `SignedDnsRecord` (opaco) |
| `names_v3` (P0.2) | clave de índice (32 B, orden de bytes) | id destino (32 B) |
| `name_reverse_v3` (P0.2) | id destino (32 B, orden de bytes) | `tag(1) ‖ nombre canónico` |
| `meta` | `&[u8]` | `&[u8]` |
| `snapshot_v1_meta` (M6b) | `u64` = 0 (slot único) | `height(8 BE) ‖ tip_hash(32)` = exactamente 40 B |
| `snapshot_v1_domains` (M6b) | `DomainId` (32 B, orden de bytes) | `DomainStateBytes` (mismo formato que `domains`) |
| `snapshot_v1_tlds` (M6b) | `TldId` (32 B, orden de bytes) | `TldStateBytes` (mismo formato que `tlds`) |
| `tld_floors_v1` (opción del propietario) | `TldId` (32 B, orden de bytes) | `u64` (piso de expiración, BE implícito de redb) |
| `snapshot_v1_floors` (M6b) | `TldId` (32 B, orden de bytes) | `u64` — copia congelada de los pisos (estado EN el snapshot, no el tip) |

Clave `meta` adicional (P0.3/P0.4):

| Clave | Valor |
|---|---|
| `verified_snapshot_v1` (P0.3) | `height(8 BE) ‖ tip_hash(32) ‖ manifest_hash(32)` = 72 B — marcador de un snapshot verificado importado (reservado, como `tip`) |

Claves de `meta`:

| Clave | Valor |
|---|---|
| `format_version` | `u64 BE` = `STORAGE_FORMAT_VERSION` (1) |
| `tip` | hash (32 B) del bloque tip |
| `tip_height` | `u64 BE` (hace `tip()` O(1)) |
| `domain_count` | `u64 BE` (contador mantenido) |
| `tld_count` | `u64 BE` (contador mantenido, M7d) |

La cabecera `hash ‖ height` de `blocks_by_height` hace cada índice
autocontenido (verificable sin leer el bloque); ambos índices se
actualizan en la misma transacción.

### Convención de génesis

El bloque génesis es determinista (constantes de protocolo) y
**nunca se guarda** como bytes. Un almacén fresco tiene `tip == (0,
genesis_hash)`; el primer bloque añadido tiene altura 1. El hash
del génesis siempre se re-deriva
(`scone_blockchain::genesis_hash()`), nunca se lee.

## Formato de codificación `DomainStateBytes` (M8b)

57 o 89 bytes según la etiqueta de versión inicial, enteros
big-endian de ancho fijo (canónico, comparable byte a byte):

```text
DomainStateBytes = tag ‖ owner(32) ‖ sequence(8 BE) ‖ registered_at(8 BE)
                   ‖ valid_until(8 BE) [‖ record_hash(32)]

tag = 0x01 : record_hash presente → exactamente 89 bytes
tag = 0x02 : record_hash ausente  → exactamente 57 bytes
```

- `registered_at` / `valid_until` (M8b): la expiración del registro
  (1 año en la reclamación, renovable, ver
  `/docs/en/technical/blockchain.md`).

Las codificaciones M7 (41/73 bytes, sin expiración) son
**ilegítimas** desde M8b: decodificación estricta → `Corrupted`.
(Un almacén M7 se regenera vía replay: `load_chain_replay`).

- `owner`: `OwnerId` (32 B);
- `sequence`: la secuencia del último `UpdateDomain` aplicado (0
  justo tras un `RegisterDomain`);
- `record_hash`: el compromiso on-chain actual del conjunto de
  registros DNS (`None` mientras no se haya aplicado ningún
  `UpdateDomain`).

Decodificación **estricta** (`DomainStateBytes::decode`): etiqueta
desconocida, longitud errónea o bytes sobrantes →
`StorageError::Corrupted`, nunca un panic.

## Formato de codificación `TldStateBytes` (opción del propietario)

Longitud variable según la variante del enum — 34 bytes para un TLD
**privativo**, 2 para un TLD **abierto** (ni secuencia ni hash de
registro):

```text
TldStateBytes = tag(0x04) ‖ flag(0x00|0x01) ‖ owner(32)? ‖ until(8)
```

- `flag` (opción del propietario, enum `TldState`): `0x00` =
  `TldState::Open` (de nadie — autoservicio + PoW, sin bytes de
  owner), `0x01` = `TldState::Private` (siguen 32 bytes de owner).
  Estrictamente canónico — cualquier otro byte → `Corrupted`. El
  invariante «abierto ⟺ sin propietario» es ESTRUCTURAL: los dos
  estados no pueden confundirse.
- **`until` (8 bytes, BE64)**: el reloj propio del TLD — el fin del
  plazo privativo (reclamación/renovación + 3 años) o el plan de
  muerte abierto (piso de 1 año al nacer, empujado por cada dominio
  vivo). Es de nuevo un atributo de la entrada (cambio de ciclo de
  vida del TLD). La tabla derivada `tld_floors_v1` (+ su copia
  congelada `snapshot_v1_floors`) sigue rastreando los pisos,
  reconstruidos en replay y restaurados al arrancar — un nodo
  reiniciado mata los mismos TLDs que un nodo continuo.

Las codificaciones M7d (33 bytes, tag `0x01`), M8b (34 bytes, tag
`0x02`, booleano `open`) y la etiqueta legada `0x03` (`until`
embebido) son **ilegítimas**: decodificación estricta →
`Corrupted` (pre-mainnet, la regeneración vía replay es la ruta de
migración).

La etiqueta existe solo para la evolución del formato: un layout
futuro sube la etiqueta y los lectores antiguos fallan con
`Corrupted` en lugar de adivinar. Decodificación **estricta**
(`TldStateBytes::decode`): etiqueta desconocida, longitud errónea,
bytes sobrantes o flag no canónico → `StorageError::Corrupted`,
nunca un panic.

## Garantías de atomicidad

`append_block_with_state` escribe en **una sola**
`WriteTransaction` de redb:

1. el bloque en `blocks_by_height` (comprobación previa: la altura
   no debe existir con contenido distinto, si no
   `StorageError::Conflict`);
2. el bloque en `blocks_by_hash`;
3. cada delta de estado de dominio (`domains`);
4. el contador `domain_count` (incrementado solo por el número de
   dominios **nuevos**; un `UpdateDomain` de un dominio existente
   no lo incrementa);
5. cada delta del registro de TLDs (`tlds`, M8b: reclamaciones,
   transferencias, abrir/cerrar) — **en la misma transacción**: un
   bloque escrito sin sus cambios de TLD es estructuralmente
   imposible;
6. el contador `tld_count` (solo TLDs nuevos);
7. cada **eliminación** (M8b): los dominios caducados por el GC
   determinista (`removed_domains`) y los TLDs revocados
   (`removed_tlds`) salen del almacén en la misma transacción,
   contadores decrementados — un reinicio nunca puede resucitar un
   registro caducado;
8. los deltas del **índice de nombres** de P0.2 (upserts y
   eliminaciones, ver abajo) — mismas garantías: una caída nunca
   deja una entrada de índice apuntando a un dominio ausente;
9. `tip` y `tip_height`.

redb hace commit estilo WAL: una caída a mitad de escritura deja el
estado coherente **anterior**. **Un bloque escrito sin su estado
es pues estructuralmente imposible** — este es el invariante clave
del hito.

Garantías adicionales:

- **Monotonía**: append a una altura ≠ `tip_height + 1` →
  `StorageError::NonMonotonicHeight`, no se escribe nada
  (comprobado *antes* de abrir la transacción de escritura);
- **Idempotencia**: re-añadir el bloque tip actual con el mismo par
  `(height, hash)` es un no-op (la ruta rápida compara altura y
  hash; a igual altura, bytes de bloque idénticos son también un
  no-op silencioso, bytes distintos un `Conflict`);
- **Base de datos corrupta**: archivo truncado/contaminado → error
  tipado (`StorageError::Database`/`Corrupted`) al abrir o al leer,
  nunca un panic (probado);
- **Versión de formato**: una `format_version` ≠ 1 →
  `StorageError::UnsupportedFormat` (se exige una ruta de migración
  explícita, nunca adivinar).

## Integración con la blockchain (`integration.rs`)

Separación de roles: [`ChainState`] en RAM = autoridad de consenso;
almacén = persistencia. La estrategia elegida (la más simple, como
está documentado): **persistir el estado en cada bloque** —
`store_block` escribe el bloque + los estados de los dominios
**tocados por ese bloque** (delta). El replay completo sigue siendo
posible y sirve de ruta de control/reparación.

- `store_block(store, chain, block, hash)`: codifica el bloque
  canónicamente, lee del estado RAM el estado final de los dominios
  tocados Y los TLDs reclamados (ya actualizado por `push_block`),
  append atómico.
- `load_chain(store)`: **sin replay** — lee tip + bloque tip
  (lecturas de bloque O(1)), restaura el estado de dominios en
  páginas de 100 (`DOMAIN_PAGE`) **y el registro de TLDs en páginas
  de 100** (`TLD_PAGE`, M7d — cerrando la ventana de reinicio: sin
  él, un nodo reiniciado olvidaba todo TLD reclamado y rechazaba
  todo `RegisterDomain` bajo ese TLD con `UnknownTld`, aunque su
  disco contuviera el estado completo). La cadena RAM solo conserva
  génesis + tip; los bloques históricos quedan en el almacén y se
  sirven desde él. Desde M8, `load_chain` reconstruye luego el
  **índice anti-replay TXID** (ver abajo).
- `load_chain_replay(store)`: reproduce todos los bloques
  almacenados uno a uno (nunca todos en RAM) con validación
  completa; el tip resultante debe igualar el tip almacenado, si no
  `Corrupted`.
- `rebuild_replay_index_from_store(store, chain, tip_height)`
  (M8): re-escanea los últimos
  `min(REPLAY_WINDOW_BLOCKS, tip_height)` bloques persistidos (un
  bloque en RAM cada vez) y reinserta cada TxId en el índice de la
  cadena **a su altura de inclusión original**, vía la API pública
  `Blockchain::rebuild_replay_index` (sin acceso a campos
  privados). Bloque ausente bajo el tip o no decodificable dentro
  de la ventana → `Corrupted` (esos bytes ya sirvieron para
  construir la cadena: un fallo aquí señala un almacén dañado).
- `touched_domains(block)`: ids de los dominios modificados por las
  transacciones del bloque, deduplicados, orden del bloque.
- `touched_tlds(block)`: ids de los TLDs mutados por el bloque
  (`RegisterTld`, `TransferTld`, `RevokeTld`, `SetTldOpen`),
  deduplicados, orden del bloque (M8b).
- `store_block_with_removals(...)`: variante de `store_block` que
  recibe también los dominios eliminados por el GC del bloque (M8b)
  para la escritura atómica descrita arriba. El relay alimenta esta
  lista desde `Blockchain::push_block_with_gc` (el resultado de la
  aplicación expone `gc_removed_domains`). **Desalojo GC de la
  caché DHT**: las entradas `dht_cache` de los dominios eliminados
  se borran en la MISMA transacción que el bloque (los deltas de
  append llevan la tabla `dht_cache`) — un registro DHT nunca
  sobrevive al dominio al que sirvió, las entradas caducadas nunca
  se acumulan para siempre (test:
  `gc_evicts_the_dht_cache_entry`).

El relay (M4) usará: `load_chain` al arrancar, luego por cada
bloque aceptado `push_block` (RAM) y luego `store_block` (disco).

## Persistencia del reorg (`rewind_to_height` + `persist_state_snapshot`)

Un reorg (cambio a una rama hermana más fuerte) no puede pasar por
el append monótono — escribir el sufijo ganador fallaría con
`NonMonotonicHeight`. La ruta dedicada (corrección
minoría-pegajosa, relay + almacén):

1. `rewind_to_height(fork_height)` — UNA transacción redb elimina
   los bloques por encima del punto de bifurcación y devuelve
   `tip`/`tip_height` (el índice de nombres y los estados vivos se
   corrigen en el paso siguiente);
2. `persist_state_snapshot(store, chain)` — reescribe las tablas
   vivas (`domains`, `tlds`, índice de nombres, pisos de TLD) desde
   la exportación del estado RAM post-reorg: las tablas vivas
   NUNCA describen la rama PERDEDORA al arrancar;
3. re-append monótono del sufijo adoptado (deltas ordinarios).

Un fallo de persistencia en esta ruta se registra y se degrada (el
almacén se cura al arrancar vía replay) — nunca fatal.

## Snapshot de estado de arranque (M6b)

`load_chain` tal como se describió sigue siendo O(tamaño del
estado): ningún bloque se reproduce, pero la tabla `domains`
completa se relee. Es la ruta de estado-persistido-por-bloque la que
hace innecesario el replay. El snapshot M6b aborda un coste
distinto: el caso en que el estado RAM debe reconstruirse **por
replay** (reparación, control, o un backend sin estado persistido
por bloque). Ofrece un arranque en **O(tip − H)** — reproduciendo
solo el sufijo por encima del snapshot.

### Escritura

- **Intervalo**: `SNAPSHOT_INTERVAL = 64` (constante pública de
  `scone-storage`). El disparador es el append del bloque
  `k·SNAPSHOT_INTERVAL + 1`: esa transacción congela el estado de
  las tablas vivas **tal como está antes de aplicar ese bloque**,
  es decir el estado canónico tras el bloque `k·SNAPSHOT_INTERVAL`.
- **Nunca en el tip**: el snapshot se sitúa pues como mucho en
  `tip − 1`, siempre estrictamente por debajo del tip en el momento
  de escribirse. Una caída justo después nunca puede dejar un
  snapshot «por encima» de la cadena persistida.
- **Atómico**: la congelación (vaciado + copia de las tablas
  `domains`/`tlds` a `snapshot_v1_*`, escritura de
  `snapshot_v1_meta`) ocurre en la MISMA transacción redb que el
  append del bloque disparador — una caída deja o el snapshot
  antiguo intacto, o el nuevo completo con su bloque (WAL de redb).
- **Slot único**: cada snapshot REEMPLAZA el anterior (tablas
  vaciadas y recopiadas) — sin acumulación, sin poda que gestionar,
  la huella en disco está acotada por ~2× el estado.
- El formato de los estados congelados es estrictamente el de
  `DomainStateBytes`/`TldStateBytes` (reutilización, sin nueva
  codificación); solo la fila `snapshot_v1_meta` es un formato
  nuevo (`height(8 BE) ‖ tip_hash(32)`, decodificación estricta:
  cualquier otra longitud → `Corrupted`).

### Lectura al arrancar (`load_chain`)

1. El tip se valida como siempre (hash recomputado desde la
   cabecera del bloque tip contra `meta["tip"]`).
2. Si `snapshot_meta()` existe, su altura `H` satisface
   `0 < H < tip_height`, y el hash RECOMPUTADO del bloque
   almacenado en `H` iguala el `tip_hash` del snapshot (**ancla**
   — un snapshot ajeno a la cadena persistida, alterado o de otro
   archivo se detecta aquí), el estado congelado se restaura en
   páginas de 100 y luego solo los bloques `H+1..=tip` se
   reproducen (validación completa, idéntica a los bloques vivos).
3. El tip alcanzado por el replay debe igualar el tip almacenado.
4. **Cualquier fallo en esta ruta (snapshot ausente, altura
   inutilizable, ancla inválida, bytes no decodificables, replay
   fallido) degrada silenciosamente a la carga completa pre-M6b**
   (leyendo las tablas vivas, sin replay) — un snapshot malo nunca
   hace fallar el arranque. La compatibilidad hacia atrás es el
   caso particular «sin snapshot»: un almacén escrito antes de M6b
   (sin las tablas `snapshot_v1_*`) carga idénticamente, siendo
   `snapshot_meta()` `None`.

El sufijo reproducido está acotado por `SNAPSHOT_INTERVAL` (como
mucho 64 bloques al dispararse + los bloques minados desde
entonces): el arranque es O(tip − H) con H garantidamente reciente.

## Índice anti-replay TXID reconstruido al arrancar (M8)

El índice anti-replay (`included`, TxId → altura de inclusión,
ventana `REPLAY_WINDOW_BLOCKS = 256` — ver
`/docs/en/technical/blockchain.md`) es una **función pura de la
cadena canónica**: nunca se persiste. Como `load_chain` no
reproduce el historial, una cadena restaurada antes de M8 arrancaba
con un índice **vacío** — un nodo reiniciado aceptaba la
re-inclusión de cualquier transacción aún dentro de su ventana de
replay (limitación documentada; las reglas de estado seguían siendo
la única salvaguarda).

M8 cierra esta ventana: tras la restauración (ruta completa O
snapshot + replay del sufijo), `load_chain` llama a
`rebuild_replay_index_from_store`, que relee los últimos
`min(REPLAY_WINDOW_BLOCKS, tip_height)` bloques persistidos — un
bloque en RAM cada vez, memoria acotada como en todo el resto — y
reinserta cada TxId a su **altura original**. Un almacén con menos
de 256 bloques reconstruye pues su índice entero.

Propiedades:

- **Decisiones idénticas al nodo vivo**: mismas entradas, misma
  regla de poda (`h > tip − 256`) — un nodo que se reinicia y un
  nodo que nunca paró rechazan los mismos replays (probado:
  recuento exacto del índice e igualdad con el índice vivo
  podado).
- **Semántica de poda sin cambios**: una transacción más antigua
  que la ventana sigue siendo admitida por el índice (y luego
  falla las reglas de estado si aplica) — exactamente como antes.
- **El snapshot NO congela el índice**: el replay del sufijo del
  snapshot reconstruye las entradas por encima de H, el
  re-escaneo M8 añade la parte de la ventana POR DEBAJO de H
  (mismo contrato que owner_keys/grace: reconstruido, nunca
  persistido).
- **Coste**: ≤ 256 lecturas puntuales de bloque al arrancar,
  O(ventana) — independiente de la edad de la cadena.
- Un bloque ausente o no decodificable **dentro de la ventana**
  hace fallar el arranque con `Corrupted` (esos bytes ya sirvieron
  para construir la cadena); fuera de la ventana, los bloques
  históricos no se tocan.

`load_chain_replay` (cero confianza) reconstruye el índice de la
misma forma mediante su replay completo — ambas rutas convergen.

## Índice persistente Nombre → Id (P0.2)

`DomainId = BLAKE3-256(DOMAIN_ID_VERSION ‖ nombre canónico)` no es
invertible: resolver un nombre (relay/DNS) exige un índice
persistente. P0.2 lo añade al almacén, con la misma disciplina
transaccional que el resto.

### Esquema

Dos tablas compañeras, escritas y borradas juntas:

```text
names_v3        key  = BLAKE3-256("SCONE-NAME-IDX-V1" ‖ nombre canónico)  (32 B)
                val  = id destino (32 B : DomainId o TldId)

name_reverse_v3 key  = id destino (32 B)
                val  = tag(0x01 dominio | 0x02 TLD) ‖ nombre canónico
```

- La clave de `names_v3` es un **hash del nombre, no el nombre en
  claro** (tablas de índice compactas, sin gossip de nombres vía
  dumps). La clave es una **función pura del nombre**: el índice es
  reconstruible en cualquier momento desde la cadena
  (`rebuild_name_index`), no se persiste metadato adicional por él.
- `name_reverse_v3` guarda el nombre (exactamente una vez) porque
  las rutas de eliminación (GC de dominios caducados, `RevokeTld`)
  solo conocen **ids**: hay que encontrar de algún modo la clave
  hasheada que borrar. La etiqueta distingue los dos espacios de
  nombres.
- Separación estricta: `resolve_name` NUNCA responde una entrada de
  TLD y `resolve_tld` nunca una de dominio (decide la etiqueta de
  la tabla inversa; un `DomainName` tiene ≥ 2 etiquetas, un TLD
  tiene 1 — ambas derivaciones de clave comparten el espacio
  `names_v3` pero no los resultados).
- El `DomainId` sigue siendo de 32 bytes, nunca reducido (regla
  dura del proyecto: identidad de consenso, resistencia a
  colisiones).

### Consistencia transaccional

Los upserts (`RegisterDomain`, `AssignDomain`, `RegisterTld` —
nombres transportados en claro por las transacciones) y las
eliminaciones (GC, revocaciones — ids desnudos) los lleva el
`StateDelta` del bloque (`name_upserts` / `name_removals`) y se
aplican en la **misma transacción redb** que las escrituras del
bloque y del estado. Una caída no puede dejar ni una entrada de
índice apuntando a un dominio ausente (sin fantasmas), ni un
dominio registrado sin su entrada. Un append fallido no escribe
nada (probado). Las variantes con nombre
`put_domain_state_with_name` / `put_tld_state_with_name` /
`remove_domain_state` ofrecen la misma atomicidad a las
herramientas de reparación.

`store_block` / `store_block_with_removals` derivan los deltas de
nombres de las MISMAS transacciones del bloque — sin parámetro
extra en la interfaz del relay.

### Reconstrucción (`RedbStore::rebuild_name_index`)

El índice es una función pura de la cadena canónica, como el índice
anti-replay M8. `rebuild_name_index` lo reconstruye desde los
**bloques persistidos** (los `DomainStateBytes` NO contienen el
nombre — solo owner/secuencia/expiración — así que la fuente de
verdad es el flujo de transacciones, donde `RegisterDomain` y
`AssignDomain` transportan el nombre canónico en claro):

1. pasada 1: barrer `blocks_by_height` génesis→tip, un bloque en
   RAM cada vez; recoger las ligaduras `nombre → id` vistas en
   `RegisterDomain`/`AssignDomain`/`RegisterTld`;
2. pasada 2: filtrar solo los destinos **vivos** (el estado
   persistido es autoritario) — un dominio registrado y luego
   GC'do/caducado no se re-indexa;
3. pasada 3: borrar + reescritura completa de ambas tablas en UNA
   transacción (caída → índice antiguo o nuevo completo).

Un bloque ausente o no decodificable → `Corrupted` (los mismos
bytes que construyeron la cadena). El coste es O(nombres distintos
vistos) en RAM (≤ 253 B por nombre) y O(bloques) en lecturas — una
herramienta de reparación, no una ruta de arranque.

### Lo que el índice NO es

- No es una autoridad: el estado RAM de la cadena sigue siendo la
  única autoridad; el índice es una caché reconstruible (una
  divergencia es un bug de transaccionalidad, detectable comparando
  `rebuild_name_index` contra él).
- No se congela en el snapshot M6b: las tablas
  `names_v3`/`name_reverse_v3` son vivas y siguen los deltas de
  cada bloque; el snapshot de estado no las embebe (reconstrucción
  por deltas al reanudar, o `rebuild_name_index` como reparación).

### Lo que el snapshot NO compromete

El estado RAM `ChainState` lleva estructuras internas no
comprometidas por consenso (claves de propietarios, ventanas de
gracia). NO se congelan: el replay del sufijo las reconstruye
aplicando las transacciones (determinista, mismo contenido que el
replay completo para la ventana pertinente). El índice anti-replay
TXID sigue el mismo contrato (M8, ver la sección anterior):
reconstruido al arrancar, nunca congelado. La igualdad bit-exacta
con `load_chain_replay` la verifica un test sobre el estado canónico
(state_root + SMT).

## Snapshot de estado VERIFICADO — trozos + manifiesto + verificación criptográfica (P0.3)

El snapshot M6b anterior acelera el arranque de un nodo que confía
en su disco. P0.3 aborda una necesidad distinta: transportar un
estado entre nodos (bootstrap de un nodo nuevo, reparación) sin
confiar en el transporte. El formato y la verificación viven en
`crates/scone-storage/src/verified_snapshot.rs` + la reconstrucción
de la raíz en
`crates/scone-blockchain/src/snapshot_verify.rs`.

### Modelo de confianza

El snapshot lleva el estado de un bloque **finalizado** — un
checkpoint firmado por el quórum del comité
(`CheckpointData.state_root`, la raíz O(1) `state_root_smt`). La
autenticidad del checkpoint (quórum de firmas Ed25519) la valida la
capa de cadena ANTES de la importación. La importación misma
verifica la adherencia snapshot ↔ checkpoint por RECOMPUTACIÓN:

1. **hash de cada página**: `blake3("SCONE-SNAP-PAGE-HASH" ‖
   codificación completa de la página)` debe igualar el hash
   anunciado por el manifiesto (un byte falsificado → rechazo);
2. **suma de entradas == contadores del manifiesto** (una página
   que falte o un contador erróneo → rechazo);
3. **raíz SMT recomputada desde las entradas importadas ==
   `checkpoint.data.state_root`** — LA comprobación criptográfica:
   el estado entero (dominios + TLDs) se re-pliega en un SMT fresco
   (`snapshot_verify::recompute_state_root`, hojas
   `SCONE-LEAF-DOM-V1`/`SCONE-LEAF-TLD-V1` exactamente como la ruta
   viva) y la raíz debe igualar la que firmó el comité. Un
   snapshot falsificado, truncado o incompleto se rechaza ANTES DE
   CUALQUIER ESCRITURA;
4. **coherencias estructurales**: `height`/`tip_hash`/`state_root`
   del manifiesto == los del checkpoint; ids estrictamente
   crecientes (sin duplicados); índices de página secuenciales;
   cotas DoS (≤ 10.000 entradas/página, ≤ 2^20 páginas,
   presupuesto total).

Fallo de verificación → `StorageError::SnapshotRejected`, el
almacén queda estrictamente sin cambios (todo se verifica antes de
la transacción de escritura).

### Formato serializado (estricto, big-endian, decodificación sin piedad)

```text
manifest = "SCONE-SNAP-MAN-V1"
        ‖ height(8) ‖ tip_hash(32) ‖ state_root(32)
        ‖ page_count(4) ‖ domain_count(8) ‖ tld_count(8)
        ‖ page_hashes(page_count × 32)

page    = "SCONE-SNAP-PAGE-V1"
        ‖ index(4) ‖ height(8) ‖ state_root(32)      ← cada página es
        ‖ entry_count(4) ‖ entries                   auto-identificativa

entry   = kind(0x01 dominio | 0x02 TLD | 0x03 piso de TLD) ‖ id(32)
          ‖ [len(2) ‖ state | floor(8 BE)]
          (state = DomainStateBytes 57/89 B o TldStateBytes 2/34 B —
           los formatos canónicos de almacenamiento existentes,
           reutilizados; el kind 0x03 (piso de vida de un TLD
           abierto, opción del propietario) tiene un formato dedicado
           SIN prefijo de longitud: id(32) ‖ floor(8 BE), tamaño fijo)
```

Orden de páginas: todas las entradas de dominio (ids crecientes),
luego todas las de TLD (ids crecientes), luego los pisos de vida
de los TLDs CONGELADOS a la altura del snapshot
(`snapshot_tld_floors_all` — no la tabla viva) — el orden de la
paginación por cursor del almacén. Cada página lleva `(index,
height, state_root del checkpoint, entries)`: una página sola sabe
a qué snapshot pertenece. Páginas de 256 entradas en la exportación
(`VERIFIED_SNAPSHOT_PAGE_ENTRIES`); cotas DoS en la importación.
Los pisos de TLD (`tld_floors_v1`) forman parte del formato
portable: el kind 0x03 los lleva, y la importación los exige
estrictamente ascendentes y apuntando a un TLD PRESENTE en el
snapshot — de lo contrario el importador nunca mataría lo que un
exportador mataría (fork silencioso, rechazado).

### Exportación (`export_verified_snapshot(store, checkpoint)`)

Serializa el estado del ÚLTIMO CHECKPOINT FINALIZADO — no el tip.
Emite dominios, TLDs, luego los pisos de vida de los TLDs
congelados a la altura del snapshot (simetría
exportación/importación: la exportación omitía históricamente los
pisos — corregido, test de regresión
`export_carries_tld_lifetime_floors`). Requisito previo: el almacén
contiene un snapshot M6b persistido (`snapshot_v1_*`)
exactamente a la altura del checkpoint (la ventana
`CHECKPOINT_KEEP`/`SNAPSHOT_INTERVAL` garantiza que tal estado
está regularmente disponible); el ancla del bloque se RECOMPUTA
desde los bytes almacenados y debe igualar
`checkpoint.data.block_hash`. Si no →
`StorageError::SnapshotUnavailable` (la exportación es una acción
explícita: esperar el próximo intervalo/checkpoint).

### Importación (`import_verified_snapshot(store, checkpoint, manifest, pages)`)

Verifica todo (ver arriba), luego escribe en UNA transacción redb:
tablas vivas `domains`/`tlds` + contadores + `tip = (H, hash del
checkpoint)`, tablas congeladas `snapshot_v1_*` ancladas al mismo
bloque, y el marcador reservado
`meta["verified_snapshot_v1"]` (`height ‖ tip_hash ‖
manifest_hash`). Destino: solo un almacén vacío (si no
`SnapshotUnavailable`).

### Coste de verificación (documentado)

Reconstrucción SMT por inserciones ordenadas: N entradas × O(40)
hashes ≈ O(N·40). ~2,1 M hashes blake3 para 100.000 dominios
(decenas de ms). PUNTUAL — una importación de bootstrap/reparación,
nunca una ruta por bloque: aceptable. La inserción ordenada (ids
crecientes) crea cada punto de bifurcación una sola vez; en
cualquier caso la raíz depende solo del contenido (garantía SMT,
fuzz diferencial en `smt.rs`).

## Bootstrap de estado — sin replay del génesis (P0.4)

Un nodo nuevo se une a la red sin reproducir el historial desde el
génesis:

```text
checkpoint finalizado (relay, validado por la capa de cadena)
        │
        ▼
snapshot verificado (manifiesto + páginas, verificación criptográfica arriba)
        │  importación → estado en H + tip (H, hash) + marcador
        ▼
bloque ancla H (pedido a la red POR HASH — append_bootstrap_anchor)
        │
        ▼
bloques recientes H+1..H+k (relay, altura creciente, append atómico
        │               ordinario — la API de almacenamiento solo exige esto)
        ▼
sincronizado — el arranque reproduce SOLO el sufijo por encima de H
```

- **API de almacenamiento expuesta al relay**:
  `append_bootstrap_anchor(H, hash, bytes)` (pone el bloque ancla:
  marcador requerido, coherencia altura/hash con el marcador,
  idempotente) luego el append atómico ordinario por altura
  creciente. La recuperación por red (por hash desde el manifiesto)
  es asunto del relay, no del almacén.
- **`load_chain`** reconoce el marcador: el historial por debajo de
  H está legítimamente ausente (nodo no-archivo). En concreto: el
  re-escaneo de la ventana anti-replay M8 se suela en H (los
  bloques por debajo de H no existen y no faltan); un tip sin
  bloque almacenado TRAS una importación (ancla aún no puesta) →
  diagnóstico tipado `SnapshotUnavailable` («el ancla debe venir de
  la red»), no corrupción.
- **Nodo archivo vs nodo de estado completo**: un nodo archivo
  (flag del operador) lo conserva todo — sirve bloques históricos
  a la red y arranca por las rutas existentes. Un nodo de estado
  completo (valor por defecto P0.4 para un nodo nuevo CONFIGURADO
  así) arranca en H: valida los bloques futuros (replay del sufijo)
  pero no puede servir el historial por debajo de H ni reproducir
  el génesis. **El comportamiento por defecto NO cambia**: esta
  ruta solo está activa si el operador dispara explícitamente la
  importación (`import_verified_snapshot` es una acción dedicada,
  nunca automática); todo nodo existente arranca idénticamente.
- **Igualdad con el nodo completo**: tras unirse al sufijo
  H+1..H+k, el estado, el tip y el `state_root_smt` son bit-exactos
  con un nodo completo al mismo tip (probado); el índice
  anti-replay cubre exactamente las txs en/por encima de H — un
  nodo no-archivo no puede conocer la ventana por debajo de H, las
  reglas de estado siguen siendo la salvaguarda (una tx duplicada
  por debajo de H falla por estado, nunca aceptada dos veces).

## Política de memoria

- **Cursores en todas partes**: `iterate_domains` acota la página a
  `max` entradas (100 por defecto) vía el `range` perezoso de redb;
  la tabla de dominios entera nunca se materializa. Las grandes
  listas (TLDs, dominios) pasan obligatoriamente por este cursor
  (`iterate_tlds`, mismo contrato, M7d).
- **Solo deltas**: `append_block_with_state` escribe solo los
  dominios y TLDs modificados por el bloque — nunca el estado
  completo;
- **Sin duplicación síncrona**: el estado RAM no se re-persiste por
  bloque; el estado en disco no se recarga por bloque;
- **Lecturas puntuales**: toda lectura abre una transacción de
  lectura corta, devuelve el valor, cierra; los `AccessGuard` de
  redb viven solo el tiempo de la copia acotada;
- **Los bloques históricos nunca en RAM**: `load_chain` carga solo
  el tip; `load_chain_replay` hace streaming altura por altura;
- **Contador mantenido**: `domain_count` es O(1), no un barrido.

## Cotas DoS

El almacén acota lo que un par hostil puede hacerle persistir o
hacerle asignar:

- **`MAX_DHT_CACHE_ENTRY` (64 KiB)**: un registro DHT mayor se
  rechaza antes de cualquier transacción de escritura por
  `put_dht_cache` → `StorageError::TooLarge` (no se escribe nada);
- **`MAX_DOMAIN_PAGE` (10.000)**: `iterate_domains` *internamente*
  acota la página pedida a `min(max, 10_000)` — un `max` excesivo
  nunca materializa más de 10.000 estados en RAM; `iterate_tlds`
  tiene la misma cota (`MAX_TLD_PAGE`, M7d);
- **Claves `meta` reservadas**: `meta_set` rechaza `tip`,
  `tip_height`, `format_version`, `domain_count`, `tld_count` y
  `verified_snapshot_v1` (P0.3) con
  `StorageError::ReservedKey` — sobrescribir la contabilidad
  interna del almacén lo corrompería en silencio.
- **Caché DHT de doble cota (LRU)**: `DHT_CACHE_MAX_ENTRIES`
  (100.000 entradas) Y `DHT_CACHE_MAX_BYTES` (2 GiB — guarda del
  caso hostil; cada entrada ya está acotada a 64 KiB por
  `MAX_DHT_CACHE_ENTRY`), la cota que se alcance primero desaloja
  por orden de clave. Los contadores (entradas + bytes) se
  mantienen en escritura — nunca un barrido para contar; la
  reconstrucción exacta ocurre solo al abrir (el único barrido
  completo, al arrancar). Accesores
  `dht_cache_entries`/`dht_cache_bytes`, cotas ajustables vía
  `set_dht_cache_bounds` (test:
  `dht_cache_respects_the_double_bound`).

## Modelo de confianza

Dos rutas de carga, dos niveles de garantía:

- **`load_chain` — confianza en el disco local**: el hash del tip
  se **recomputa** desde la cabecera del bloque tip y se compara
  con el `meta["tip"]` almacenado (divergencia →
  `StorageError::Corrupted`, nunca una cadena restaurada sobre un
  hash falsificado), y los estados de dominios se decodifican
  estrictamente. Sin embargo, los bloques históricos **no se
  reproducen**: los estados persistidos se toman a valor aparente.
  Es el arranque rápido de un nodo que confía en su propio disco
  (escrito solo por ese mismo nodo, atómicamente). Desde M6b, esta
  ruta usa primero el **snapshot de estado** cuando es utilizable
  (ver abajo): el ancla del snapshot se recomputa y el sufijo por
  encima de H se revalida como bloques vivos — la parte reproducida
  es pues cero-confianza, la base congelada confianza-disco.
- **`load_chain_replay` — cero confianza**: todos los bloques se
  releen y revalidan uno a uno (validación completa, idéntica a los
  bloques vivos); el tip resultante debe igualar el tip
  almacenado. Es la herramienta de control/reparación.

**Recomendación**: un nodo estricto (o un nodo cuyo archivo de
almacenamiento pudo ser manipulado — disco compartido, restauración
de un backup dudoso) debe arrancar vía `load_chain_replay`, no
`load_chain`.

## Concurrencia

redb 4.x permite **un solo objeto `Database` por archivo y por
proceso** (el doble `open` simultáneo falla con
`DatabaseAlreadyOpen`). El modelo soportado: un `RedbStore` clonado
(`Arc<Database>`) compartido entre hilos — escritores serializados
por `&mut self` + el bloqueo interno de redb, lectores concurrentes
(transacciones de lectura snapshot). Tras soltar todos los
handles, el archivo puede reabrirse.

## Tests (`tests/redb_store.rs` + módulos `#[cfg(test)]`)

Cada test usa su propio tmpfile redb (`tempfile`). Cobertura:

- roundtrip de bloques génesis→N, reinicio, relectura, replay;
- M7d: registro de TLDs persistido por delta atómico, restaurado al
  reiniciar (un `RegisterDomain` bajo un TLD reclamado antes del
  reinicio sigue siendo admisible — ventana cerrada), contadores
  exactos al reabrir, cursor `tlds` ascendente, un append fallido
  no deja TLD, clave reservada `tld_count` rechazada;
- atomicidad: un append fallido no deja nada (tip, dominios,
  índice intactos); deltas duplicados contados una vez;
- cursores de dominios: paginación 100/7, orden ascendente de
  `DomainId`, arranque estrictamente tras un id existente o
  ausente, `max = 0`;
- contador `domain_count` exacto
  (inserción/actualización/reapertura);
- roundtrip de caché DHT + sobrescritura;
- concurrencia: lecturas desde dos hilos sobre handles clonados;
- base de datos corrupta: archivo truncado / basura → error tipado,
  sin panic (`catch_unwind`);
- M6b: snapshot escrito en el intervalo, estrictamente por debajo
  del tip; arranque desde snapshot = replay completo bit-exacto
  (state_root + SMT, estado lógico completo, tip); snapshot
  alterado (tip_hash corrupto, estado no decodificable, altura >
  tip, meta de longitud errónea) → fallback silencioso a la carga
  completa, misma cadena final; sufijo reproducido < intervalo;
  sustitución del snapshot (sin acumulación) al cruzar el
  intervalo; almacén pre-M6b sin tablas de snapshot carga
  idénticamente;
- M8: índice anti-replay reconstruido al arrancar — una tx
  re-incluida dentro de la ventana se rechaza (`TxReplay`) tras la
  restauración; una tx más antigua que la ventana se admite (poda
  sin cambios, deciden las reglas de estado); recuento exacto ==
  índice vivo (== `load_chain_replay`) para un almacén más corto y
  más largo que la ventana; arranque con snapshot: la parte de la
  ventana por debajo de H la reconstruye el re-escaneo, la de arriba
  el replay del sufijo;
- P0.2: índice de nombres — `resolve_name == DomainId::from_name`
  para 120 dominios (vivo, tras reinicio, tras borrado +
  reconstrucción), `resolve_tld` para el TLD reclamado; el GC de un
  dominio caducado elimina la entrada (sin fantasmas), antes y
  después de la reconstrucción; separación estricta de los
  espacios de nombres dominio/TLD; un append fallido no deja
  entrada de índice; la clave del índice es la derivación BLAKE3
  documentada;
- P0.3/P0.4 (`tests/verified_snapshot.rs`): exportación →
  importación → arranque == replay completo (estado, tip,
  `state_root_smt` bit-exacto; el estado importado es el del
  bloque finalizado, no el tip); página falsificada en un byte →
  rechazo antes de cualquier escritura (almacén estrictamente sin
  cambios); página ausente → rechazo; `state_root` erróneo
  (manifiesto ↔ checkpoint, luego raíz recomputada ≠ raíz firmada)
  → rechazo; contadores / alturas / tips incoherentes / ids
  duplicados → rechazo; la importación exige un almacén vacío; la
  exportación exige un snapshot persistido a la altura del
  checkpoint; contrato del ancla (`append_bootstrap_anchor`:
  marcador requerido, coherencia, idempotencia); formato
  serializado estricto (bytes sobrantes, etiquetas, cotas DoS);
  snapshot en H y luego bloques H+1..H+k unidos → cadena idéntica
  al nodo completo (estado/tip/raíz), arranque sin historial por
  debajo de H, índice anti-replay = exactamente las txs en/por
  encima de H.

[`ChainState`]: ../../../crates/scone-blockchain/src/state.rs
