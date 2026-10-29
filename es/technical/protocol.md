# Protocolo

Documento de referencia del protocolo Scone: formato binario,
codificación canónica, transacciones, registros DNS, bloques,
mensajes P2P, límites y compatibilidad de versiones. La
implementación de referencia es el crate
[`scone-protocol`](../../../crates/scone-protocol).

Los elementos marcados **(provisional)** no son definitivos.

## Versión del protocolo

- constante: `PROTOCOL_VERSION: u32 = 1` (`scone-protocol`);
- transportada por `Hello.version` (handshake) y
  `BlockHeader.version`;
- reglas de compatibilidad:
  - versión recibida **mayor** que la local → rechazar
    (`ProtocolError::UnsupportedVersion`); el nodo no adivina el
    formato de una versión que no conoce;
  - versión `0` → inválida;
  - solo hay un juego de formatos: los bytes de versión de
    transacción (`0x01`) y de bloque deben coincidir
    **exactamente** con los valores locales;
- sin división mayor/menor en esta etapa: un solo entero,
  incrementado ante cualquier ruptura del formato wire.

## Principios del formato binario

- binario, nunca JSON para datos de protocolo;
- determinista y canónico: dos objetos lógicamente iguales producen
  **exactamente los mismos bytes**;
- análisis secuencial, sin asignaciones donde sea posible (borrows
  de bytes);
- toda entrada de red es no confiable: longitudes y contadores se
  comprueban **antes** de cualquier asignación; la decodificación
  nunca hace panic.

### Endianness

- todos los enteros son varints LEB128 (grupos de 7 bits, el menos
  significativo primero): la endianness nativa no aplica;
- direcciones IPv4/IPv6 e identificadores de 32 bytes: arrays de
  bytes crudos, sin prefijo de longitud;
- no hay ningún entero ancho de tamaño fijo en el formato: una sola
  regla de codificación de enteros.

### Varint (LEB128 mínimo)

Cada byte transporta 7 bits de datos; el bit alto vale 1 en todos
los bytes salvo el último. Un `u64` cabe en 1 a 10 bytes.

| valor | bytes (hex) |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16.384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

La codificación debe ser **mínima**: las formas sobredimensionadas
se rechazan en decodificación (`InvalidVarint`). Es la condición
necesaria para que un valor tenga una sola secuencia de bytes
válida (codificación canónica). Rechazos: `80 00` (no mínimo),
`ff×9 02` (desbordamiento de u64), `ff×10` (demasiado largo).

### Cadenas y bytes

- `bytes`: varint de longitud + bytes crudos;
- `str`: ídem, con decodificación UTF-8 obligatoria;
- toda longitud está acotada (ver [Límites](#límites)) y
  comprobada antes de leer.

### Identificadores

| Tipo | Representación wire |
|---|---|
| `DomainId` | 32 bytes crudos |
| `TldId` | 32 bytes crudos |
| `OwnerId` | 32 bytes crudos |
| `PublicKeyRef` | 32 bytes crudos |
| `RecordHash` | 32 bytes crudos |
| `BlockHash` | 32 bytes crudos |
| `MerkleRoot` | 32 bytes crudos |
| `DomainName` | `str` (≤ 253 bytes, revalidado por `scone-core`) |

Las derivaciones (`BLAKE3-256` + prefijos `SCONE-DOMAIN-V1`, etc.)
siguen definidas por `scone-core` (ver
`/docs/en/general/naming.md`).

## Transacciones

```text
Transaction = disc u8 || version u8 || payload
```

| Discriminante | Tipo |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x2A` | `RegisterTldPow` |
| `0x52` | `UpdateDomain` |
| `0x57` | `RenewTldPow` |
| `0x84` | `RegisterDomainPow` |
| `0x93` | `RegisterTld` |
| `0xC3` | `RenewDomainPow` |

Cada discriminante es una constante opaca arbitraria (sin
secuencia lógica; `0x00` nunca válido) — tabla normativa completa
en `/docs/en/technical/transactions.md`.

El byte de `version` (valor actual `0x01`) sigue al discriminante:
cualquier otro valor se rechaza explícitamente
(`UnsupportedVersion(value)`). La mayoría de las transacciones van
firmadas (Ed25519) — EXCEPTO las cuatro formas sin firma
pow-to-owner (`0x2A`, `0x57`, `0x84`, `0xC3`): la prueba minada ES
la autorización, y su reto liga la clave del beneficiario y la
red. Especificación completa, payload firmado y reglas de
validación en **`/docs/en/technical/transactions.md`**
(normativo).

Desde el cambio de wire de 2026-09, **las transacciones ya no
llevan campo `network`**: el `network_id` (`str` ≤ 16) se inyecta
en el **payload de firma** (formas firmadas) o en el **reto PoW**
(formas pow-to-owner) — una tx hecha para una red nunca se
traslada a otra (ver `/docs/en/technical/blockchain.md`).

### REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

`RegisterDomain` lleva el **nombre canónico en claro**, seguido de
su `DomainId` — la coherencia `domain_id == DomainId(name)` se
recomprueba en decodificación (recomputada, nunca confiada). `proof`
lleva el PoW de registro (20 bytes: nonce + época + dificultad —
salt de época `epoch = height/144` del bloque aplicante, verificado
ESTRICTAMENTE en aplicación contra la constante de red y la época
del bloque; rechazo tipado `ProofEpochMismatch` en caso de
desajuste — M8b, ver `transactions.md`). `public_key` es la clave
Ed25519 del firmante (exactamente 32 bytes); `owner` debe ser la
derivación BLAKE3 de esa clave (recomputado, nunca confiado);
`signature` es exactamente 64 bytes.

### REGISTER_TLD (0x93)

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

Reclamación de un TLD (registro separado del registro de dominios,
M7): el TLD se **transporta en claro** igual que el nombre de un
dominio (M8b — el reto PoW deriva del nombre), y `tld_id` es la
derivación `SCONE-TLD-V1` de ese nombre, disjunta de todo
`DomainId` por construcción. Reglas de estado (M8b): TLD libre + no
reservado por ICANN (`IcannTldReserved` — los TLDs raíz de la
ICANN no son reclamables por nadie) + PoW a la dificultad **por
longitud** de la etiqueta (curva `pow_length` — ver
`blockchain.md`); lo reclamado queda cerrado (solo asignación).

### UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

Invariante recomprobado en decodificación: `sequence > 0` y
`owner` == derivación de `public_key`. Sin `timestamp`: el orden
pasa por `sequence`. La transacción lleva solo referencias
compactas — el contenido DNS completo queda en el DHT (ver abajo).

## Registros DNS (lado DHT)

Los registros DNS completos nunca entran en la blockchain: se
intercambian vía el DHT (mensajes `GetRecord`/`Record`) y se
comprometen on-chain solo por su `RecordHash`.

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

Los códigos de tipo son los **códigos DNS reales** (IANA). El TTL
(segundos, varint) sigue al código de tipo y cubre todo el
registro: cotas `MIN_TTL = 1` .. `MAX_TTL = 2.592.000` (30 días;
por defecto `DEFAULT_TTL = 86.400`).

| Código | Tipo | Payload |
|---|---|---|
| 1 | A | 4 bytes |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`, `name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16 bytes |
| 33 | SRV | `priority.v`, `weight.v`, `port.v`, `target(name)` |
| 257 | CAA | `flags u8`, `tag` (1–15 B), `value` (`str` ≤ 4096) |
| 32768 | GEOROUTE | GeoRoute (ver abajo) |
| otro | Unknown | code.v + `bytes` ≤ 4096 |

Ejemplo: `A 192.0.2.1` (TTL 86.400) → `01 80 a3 05 c0 00 02 01`
(tipo, TTL varint, dirección).

Un `Unknown` que lleve un código conocido se rechaza en
codificación (la decodificación nunca puede producir uno).

#### GEOROUTE (GeoDNS)

Tipo privativo de Scone (rango no IANA). Una regla de enrutado
geográfico:

```text
GeoRoute  = scope || count.v(≤ 32) || endpoint…
scope     = 0x00                    -- mundo
          | 0x01 continent.v        -- ONU M49
          | 0x02 country.v          -- ISO 3166-1 numérico
          | 0x03 region.v           -- código de región
          | 0x04 asn.v              -- número de AS
          | 0x05 prefix.addr[16] || prefix.len u8   -- prefijo IP (normalizado)
endpoint  = target || priority.v || weight.v
target    = 0x00 octets[4]          -- IPv4 binario
          | 0x01 octets[16]         -- IPv6 binario
```

**Regla canónica**: los endpoints de un mismo ámbito se codifican
**ordenados** (prioridad ascendente, luego peso descendente, luego
target ascendente); el orden se aplica en codificación y un wire
desordenado se rechaza en decodificación (`NonCanonical`), igual
que los conjuntos de registros. Los duplicados completos de
endpoint se rechazan.

**Validación**: `priority > 0`, `weight ≤ 100` (0 = endpoint
inactivo), ≤ 32 endpoints por ámbito (`GEO_MAX_ENDPOINTS`). Un
código de país/continente desconocido para la tabla local
(`continent_of_country`) se **acepta** a nivel wire: la tabla del
resolvedor puede estar más al día que la del publicador. El TTL es
el del `RecordData` padre (sin cambios).

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**Regla canónica**: un conjunto de registros es una colección sin
orden. Se codifica **ordenada por orden lexicográfico de la
codificación individual de cada registro**, estrictamente
creciente (duplicados rechazados). Un wire desordenado se rechaza
en decodificación (`NonCanonical`). El orden del `Vec` original no
se transporta.

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

El hash cubre **solo el contenido del registro**; la firma del
propietario se verifica aparte sobre los mismos bytes canónicos.
*(Decisión: el compromiso no cubre la firma — una versión anterior
de este documento mencionaba el «SignedDnsRecord completo»; acoplar
el hash a la firma no aportaba nada y ataba el compromiso al
esquema de firma.)*

## Bloques

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65.536)
```

- `version`: versión del protocolo (reglas anteriores);
- `height`: altura (génesis = 0);
- `prev_hash`: hash del bloque anterior (ceros para el bloque
  génesis);
- `tx_root`: compromiso sobre la lista de transacciones (árbol
  Merkle definido por `scone-blockchain`, ver
  `/docs/en/technical/blockchain.md`);
- `timestamp`: información de orden (Unix, segundos);
- `consensus`: payload del productor (`SCONE-BLOCK-V1 ‖ pk[32] ‖
  sig[64]`, ver `blockchain.md`) — el génesis de una red lleva el
  `network_id` crudo, y un génesis de asignación (kickstarter de la
  mainnet) puede llevar un manifiesto `ALLOC-V1` (≤ 60 KiB);
  acotado por `MAX_CONSENSUS_LEN` (64 KiB).

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

Análisis secuencial; el contador está acotado **antes** de asignar;
cualquier truncamiento se rechaza. A diferencia de los conjuntos de
registros, el orden de las transacciones es **significativo**
(orden de consenso) y se conserva exactamente.

El `BlockHash` (hash de la cabecera canónica) y el `tx_root` los
computa `scone-blockchain` sobre las codificaciones aquí definidas
(fórmulas exactas en `/docs/en/technical/blockchain.md`).

## Mensajes P2P

```text
Message = disc u8 || payload
```

El transporte (framing, libp2p…) queda fuera de alcance; una
trama de transporte no debería superar `MAX_MESSAGE_LEN` (1 MiB).

| Disc | Mensaje | Dirección | Payload | Límite | Respuesta esperada |
|---|---|---|---|---|---|
| `0x01` | `Hello` | ambos | `version.v` | — | `Hello` |
| `0x02` | `Ping` | ambos | — | — | `Pong` |
| `0x03` | `Pong` | ambos | — | — | — |
| `0x04` | `GetBlock` | ambos | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | ambos | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | ambos | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | ambos | `Transaction` | — | — |
| `0x08` | `GetRecord` | ambos | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | ambos | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | ambos | `Checkpoint` | firmantes ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | ambos | — | — | ventana de checkpoints |
| `0x0c` | `GetStatus` | ambos | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | ambos | estado del nodo (red, tip, altura, NAT, dominios, mempool, pares, finalidad, operadores frescos) | — | — |
| `0x0e` | `LookupName` | ambos | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | ambos | estado on-chain del nombre + TLD, incl. la generación CAS (contadores de mutaciones, registered_at) | — | — |
| `0x10` | `GetOwnedNames` | ambos | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | ambos | dominios + TLDs vivos del propietario | — | — |
| `0x12` | `Error` | respuesta | mensaje de error a nivel de aplicación (`str`) | — | — |

### Checkpoint (M5)

Un checkpoint agregado va al wire como: tag `SCONE-CKPT-V1` ||
cuerpo `CheckpointData` (116 B, LE — idéntico al final de la
preimagen de firma: `epoch(8) height(8) block_hash(32)
prev_checkpoint_hash(32) state_root(32) recovery(4)`) ||
`count.v` || `pk[32] ‖ sig[64]` por firmante, **ordenados por pk
ascendente, sin duplicados**, `count ≤ 64`
(`MAX_CHECKPOINT_SIGNERS`). Decodificación estricta: truncamiento,
tag erróneo, pk no canónico, duplicados o desorden = rechazo
(`None`), nunca un panic (fuzz de corrupción probado).

`GetCheckpoints` pide la ventana de finalidad; la respuesta es la
ventana `checkpoint_window()` (como mucho `CHECKPOINT_KEEP`
checkpoints, del más antiguo al más reciente) vía uno o más
mensajes `Checkpoint`.

El `TxId` está ahora definido por `scone-blockchain`
(`BLAKE3-256("SCONE-TX-V1" || canonical(Transaction))`, ver
`/docs/en/technical/blockchain.md`). El mempool y el anti-replay
por ventana (`TxReplay`, `REPLAY_WINDOW_BLOCKS = 256`) están
entregados en el relay (M4); solo queda aplazado `GetTransaction`
(su definición depende de las decisiones de consenso restantes).

## Reparto blockchain / DHT

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (completo, firmado)
sequence
record_hash
```

La cadena dice *quién* posee y *cuál es el estado actual*
(secuencia + record_hash). El DHT proporciona *el contenido*. Un
registro DHT se acepta si y solo si:

1. su firma es válida para el propietario registrado en la cadena;
2. su `sequence` es coherente con (o mayor que) la de la cadena;
3. el `RecordHash` recomputado sobre su codificación canónica
   coincide con el `record_hash` de la cadena.

## Límites

| Constante | Valor | Rol |
|---|---|---|
| `MAX_NAME_LEN` | 253 | longitud de un nombre |
| `MAX_TLD_LEN` | 63 | longitud de un TLD |
| `MAX_TXT_LEN` | 4096 | cadena TXT |
| `MAX_PROOF_LEN` | 256 | prueba de registro |
| `MAX_SIGNATURE_LEN` | 128 | firma (Ed25519 = 64) |
| `MAX_RECORDS_PER_SET` | 256 | registros por conjunto |
| `MAX_DHT_RECORD_LEN` | 64 KiB | registro DNS firmado codificado (payload DHT: wire, almacenamiento, gossip) — cota total comprobada en codificación Y antes de cualquier decodificación; 15–30× por encima de cualquier conjunto realista (SPF+DKIM+DMARC ≈ 2–5 KiB), 16× por debajo del tope de transporte. |
| `GEO_MAX_ENDPOINTS` | 32 | endpoints por ámbito GeoRoute |
| `MAX_UNKNOWN_DATA` | 4096 | datos de un tipo desconocido |
| `MAX_TXS_PER_BLOCK` | 4096 | transacciones por bloque |
| `MAX_CONSENSUS_LEN` | 64 KiB | payload de consenso de una cabecera (ampliado para el manifiesto de génesis `ALLOC-V1` — kickstarter de la mainnet) |
| `MAX_BLOCKS_PER_REQUEST` | 128 | bloques por `GetBlocks` |
| `MAX_MESSAGE_LEN` | 1 MiB | trama de transporte (recomendado) |

Cualquier exceso → `LimitExceeded`, detectado antes de asignar: un
par malicioso no puede provocar una asignación desproporcionada en
decodificación.

## Codificación canónica

Regla fundamental:

```text
objeto canónico → codificación binaria canónica → hash / firma
```

- solo varints mínimos;
- conjuntos de registros ordenados estrictamente crecientes;
- un objeto lógicamente igual tiene exactamente **una** codificación
  válida;
- la decodificación rechaza las formas no canónicas
  (`NonCanonical`), de modo que no existen múltiples codificaciones
  válidas para el mismo objeto.

## Gestión de errores

`ProtocolError` (`scone-protocol::error`):

| Variante | Significado |
|---|---|
| `Truncated` | la entrada terminó en medio de un valor |
| `InvalidVarint` | varint no mínimo, desbordado o demasiado largo |
| `IntegerOutOfRange` | el entero no cabe en su tipo destino |
| `UnsupportedVersion` | versión cero o superior a la local |
| `UnknownDiscriminant` | discriminante de transacción/mensaje desconocido |
| `LimitExceeded` | longitud o recuento más allá de un límite |
| `InvalidUtf8` | cadena no UTF-8 |
| `TrailingBytes` | bytes restantes tras un valor completo |
| `Validation` | violación de un invariante de `scone-core` |
| `NonCanonical` | codificación válida por campos pero no canónica |

La decodificación nunca hace panic con datos de red.

## Firmas

- esquema: **Ed25519** (RFC 8032) vía `scone-crypto`
  (`ed25519-dalek`, verificación `verify_strict` — firmas
  maleables y claves débiles rechazadas);
- **transacciones de cadena**: cada `RegisterDomain` /
  `RegisterTld` / `UpdateDomain` embebe `public_key` (32 B) +
  `signature` (64 B) sobre el payload
  `"SCONE-TX-SIG-V1" || canonical_encode(tx_without_signature)`;
  especificación normativa completa:
  `/docs/en/technical/transactions.md`;
- **registros DHT** (`SignedDnsRecord`): el payload firmado es la
  codificación canónica del `DnsRecord`; la verificación cruza
  `record + owner + signature` con la identidad registrada en la
  blockchain;
- regla transversal: `owner` y todos los hashes/compromisos se
  **recomputan**, nunca se toman de un valor proporcionado.

## Reglas de consenso

Entregado (descrito en `/docs/en/technical/blockchain.md`):
productor de bloques firmado (payload `SCONE-BLOCK-V1`, comité
elegido por checkpoint), finalidad por checkpoints agregados
(`SCONE-CKPT-V1`, elección de rama en el piso de finalidad), PoW de
registro por longitud del nombre. Temas pendientes: tasas
antispam finas, rotación de la clave del propietario.
