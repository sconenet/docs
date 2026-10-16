# Transacciones firmadas (formato v1)

Documento de referencia **normativo** del formato de transacción
firmada. Especifica los campos, el orden wire exacto, los tamaños,
el payload firmado y las reglas de validación, para permitir una
implementación independiente sin leer el Rust. La implementación de
referencia es [`scone-protocol`](../../../crates/scone-protocol)
(codificación) y
[`scone-blockchain`](../../../crates/scone-blockchain)
(validación).

`PROTOCOL_VERSION` = **1**. Solo hay un juego de formatos:
cualquier byte de versión distinto de `0x01` (en una transacción) o
diferente de la versión local (en un bloque) se rechaza con
`UnsupportedVersion`.

## Visión general

```text
Transaction = disc u8 || version u8 || payload
```

| Campo | Tamaño | Rol |
|---|---|---|
| `disc` | 1 B | tipo de transacción (tabla abajo) |
| `version` | 1 B | **exactamente `0x01`** (versión del formato tx) |

**Las transacciones ya no llevan campo `network` en el wire**
(cambio de 2026-09): el `network_id` (`str` ≤ 16, ASCII
`[a-z0-9-]` — p. ej. `scone-testnet`, 13 bytes) vive **solo en el
payload de firma** — el firmante lo inyecta ahí como primer campo
(ver [Payload firmado](#payload-firmado)) — y permanece por red en
el digest del PoW. Una transacción **firmada** para una red nunca
verifica en otra: su firma falla (`InvalidSignature`) contra el
payload recomputado por la cadena, que inyecta su propia red (ver
`/docs/en/technical/blockchain.md` — redes y génesis).

### Discriminantes (tabla normativa)

Cada discriminante es una **constante opaca arbitraria y fija**:
los valores se eligen deliberadamente distintos, para que el wire
**nunca** sugiera un orden lógico entre tipos (sin secuencia
0x01/0x02/0x03…). `0x00` **nunca** es un tipo válido (byte de
corrupción típico). Los tipos futuros tomarán cualquier valor libre
no usado, documentado de una vez por todas en esta tabla.

| Discriminante | Tipo | Rol |
|---|---|---|
| `0x21` | `RegisterDomain` | reclamación de un dominio |
| `0x52` | `UpdateDomain` | nueva versión de los datos DNS de un dominio |
| `0x93` | `RegisterTld` | reclamación de un dominio de nivel superior (M7) |
| `0x47` | `TransferTld` | transferencia de un TLD a un nuevo propietario (M8a) |
| `0x6B` | `RevokeTld` | abandono de un TLD (M8a) |
| `0xB8` | `SetTldOpen` | apertura **irreversible** de un TLD al autorregistro (unidireccional, opción del propietario; `open=false` es un rechazo tipado) |
| `0xD4` | `AssignDomain` | asignación directa de un dominio por el propietario del TLD (M8a) |
| `0x3C` | `RenewDomain` | extensión del registro de un dominio (M8a; lleva el nombre canónico desde la opción del propietario — un TLD abierto sigue la expiración) |
| `0x8F` | `RenewTld` | renovación del plazo de 3 años de un TLD PRIVATIVO (PoW igual a la creación) |
| `0x15` | `SlashTx` | prueba de equivocación de un anchor + expulsión (M9) |
| `0xE2` | `TransferDomain` | transferencia de un dominio a un nuevo propietario (M8c) |
| `0x71` | `Heartbeat` | prueba de vida de un operador (RFC de prueba de servicio) |
| `0xA9` | `SetTldOperator` | delegación de la operación de un TLD privativo (RFC de prueba de servicio) |
| `0x5E` | `SetDomainOperator` | delegación de la operación de un dominio (RFC de prueba de servicio) |

Cualquier byte de versión distinto de `0x01` se rechaza
explícitamente (`UnsupportedVersion(valor recibido)`): un formato
desconocido u obsoleto nunca se analiza por accidente.

## SLASH_TX (0x15, M9)

Prueba on-chain de que un anchor firmó dos **checkpoints
conflictivos en la misma época** (equivocación). Cualquiera puede
enviarla — la prueba es criptográfica y autocontenida:

```text
offender[32] | evidence_a (CheckpointData 116 B)
  | sig_a[64] | evidence_b (CheckpointData) | sig_b[64]
  | reporter_pk[32] | reporter_signature[64]
```

- `validate()` comprueba la estructura: misma época,
  `signing_hash`es distintos, dos firmas distintas, firmas del
  denunciante sobre el payload canónico;
- reglas de estado (journaled, revertible en reorg): AMBAS firmas
  del acusado deben verificar contra los DOS signing hashes
  distintos en la misma época; el acusado debe estar en el pool
  elegible (`SlashOffenderNotInPool` en caso contrario); luego
  **expulsión** — la clave se elimina del pool PoS de forma
  persistente (lista negra en estado);
- sanción mínima: los dominios/TLDs del acusado NO se confiscan
  (decisión M9 documentada);
- el mismo checkpoint dos veces = no es equivocación (rechazado);
  épocas distintas = rechazado; firma falsificada = rechazado.

## REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

| Campo | Tipo | Restricciones |
|---|---|---|
| `name` | `str` ≤ 253 | nombre canónico del dominio reclamado, **transportado en claro** (ver abajo) |
| `domain_id` | 32 bytes crudos | **debe** igualar `DomainId(name)` (recomputado, nunca confiado) |
| `owner` | 32 bytes crudos | **debe** ser la derivación de `public_key` (ver abajo) |
| `timestamp` | varint | información de orden (Unix, segundos) |
| `proof` | `bytes` ≤ 256 | payload del PoW de registro (ver [abajo](#prueba-de-trabajo-de-registro-pow)) |
| `public_key` | 32 bytes crudos | clave pública Ed25519 del firmante (RFC 8032) |
| `signature` | **exactamente 64 bytes** | firma Ed25519 del payload firmado |

### El nombre viaja en la transacción

`RegisterDomain` transporta el nombre canónico en claro, seguido de
su `DomainId`. La coherencia `domain_id == DomainId(name)` se
recomprueba en codificación, decodificación y validación (defensa
en profundidad): un `RegisterDomain` cuyo id no coincide con el
nombre transportado es inválido. El `DomainId` sigue siendo la
identidad de consenso (claves de cadena y DHT); el nombre
transportado hace la cadena autodescriptiva y cubre la reclamación
entera en la firma.

## REGISTER_TLD (0x93)

Reclamación de un dominio de nivel superior (registro de TLDs, M7):

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) open(0x00|0x01) public_key[32] signature[64]
```

| Campo | Tipo | Restricciones |
|---|---|---|
| `tld` | `str` ≤ 63 | nombre canónico del TLD **transportado en claro** (M8b: el reto PoW deriva del nombre, igual que `RegisterDomain` transporta el suyo) |
| `tld_id` | 32 bytes crudos | identidad del TLD: `BLAKE3-256("SCONE-TLD-V1" || tld)` — **debe** ser la derivación de `tld` (recomputada) |
| `owner` | 32 bytes crudos | derivación de `public_key` |
| `timestamp` | varint | información de orden (Unix, segundos) |
| `proof` | `bytes` ≤ 256 | payload del PoW de registro (ver [abajo](#prueba-de-trabajo-de-registro-pow)) |
| `open` | `0x00\|0x01` | nacimiento **abierto**: el TLD aterriza directamente en `TldState::Open` (nadie, autoservicio + PoW, un solo PoW a la tarifa `open_tld_pow_difficulty` — más barato que el privativo: recurso común, sin asiento PoS ni exclusividad); nacimiento **privativo**: comportamiento histórico (propietario, asiento PoS, `tld_pow_difficulty`) |
| `public_key` | 32 bytes crudos | clave pública Ed25519 del firmante |
| `signature` | **exactamente 64 bytes** | firma Ed25519 del payload firmado |

El espacio de los `TldId` es **disjunto** del espacio de los
`DomainId` (prefijos de derivación distintos — ver
`/docs/en/general/naming.md`): registrar el TLD `uip` (una
reclamación sobre el espacio `*.uip`) nunca puede usurpar la
identidad de un dominio, ni al revés. Regla de estado en
aplicación: el TLD debe estar libre (registro separado del registro
de dominios).

## La familia M8a — formatos wire

Cinco tipos completan el registro de TLDs y el ciclo de vida de los
dominios. Formatos wire normativos desde M8a; **reglas de estado
completas desde M8b** (ver `/docs/en/technical/blockchain.md`):
propietario del TLD requerido, PoW dondequiera que una reclamación
sea en autoservicio, expiraciones y gracia en el lado de los
dominios. Ningún payload lleva campo `network` en el wire (vive
solo en el payload de firma).

### TRANSFER_TLD (0x47)

```text
tld_id[32] owner[32] new_owner[32] public_key[32] signature[64]
```

| Campo | Tipo | Restricciones |
|---|---|---|
| `tld_id` | 32 bytes crudos | el TLD transferido |
| `owner` | 32 bytes crudos | derivación de `public_key` (propietario actual / firmante) |
| `new_owner` | 32 bytes crudos | identidad del destinatario (opaca: cualquier valor es sintácticamente válido; el enlace a las claves que pueden gastarlo es una regla de estado M8b) |
| `public_key` | 32 bytes crudos | clave pública Ed25519 del firmante |
| `signature` | **exactamente 64 bytes** | firma Ed25519 del payload firmado |

Sin timestamp ni secuencia: un TLD tiene exactamente un propietario
a la vez; la primera transferencia aplicada gana (solo el orden de
la cadena).

### REVOKE_TLD (0x6B)

```text
tld_id[32] owner[32] public_key[32] signature[64]
```

Abandono voluntario de un TLD (reclamable después con un
`RegisterTld` fresco). Firmado por el propietario actual;
deliberadamente sin timestamp ni secuencia — un revoke es one-shot
contra el estado actual.

### SET_TLD_OPEN (0xB8)

```text
tld_id[32] owner[32] open u8 open_proof(proof) public_key[32] signature[64]
```

| Campo | Tipo | Restricciones |
|---|---|---|
| `open` | 1 byte | **estrictamente `0x00` o `0x01`** (forma canónica; cualquier otro valor → error de decodificación, nunca coerción) |
| `open_proof` | `proof` (varint de longitud + bytes) | campo **firmado**; obligatorio y verificado cuando `open = 0x01` (PoW de apertura, `open_tld_pow_difficulty` de la red, reto `SCONE-TLD-OPEN-V1 ‖ tld_id`); vacío cuando `open = 0x00` (cerrar es gratis) |

Un TLD **cerrado** = solo asignación (dominios creados
exclusivamente por el propietario del TLD vía `AssignDomain`); un
TLD **abierto** = cualquiera puede hacer el PoW de registro y
reclamar un dominio libre (`RegisterDomain`). **Abrir** cuesta una
prueba de trabajo adicional (más ligera que la reclamación del TLD:
~3 días en mainnet frente a ~1 semana), pagada una vez por el
propietario en `SetTldOpen` — la prueba viaja en el campo firmado
`open_proof` y no puede sustituirse tras la firma.

### ASSIGN_DOMAIN (0xD4)

```text
name(str ≤ 253) domain_id[32] owner[32] assignee[32] public_key[32] signature[64]
```

Espejo de `RegisterDomain`: el nombre canónico viaja en claro y
`domain_id` debe igualar `DomainId(name)` (recomputado en
decodificación, codificación y validación). Firmado por el
**propietario del TLD del nombre** (comprobado en aplicación,
M8b); `assignee` se convierte en el primer propietario del dominio
(opaco, como `new_owner`).

### RENEW_DOMAIN (0x3C)

```text
name(str ≤ 253) domain_id[32] owner[32] valid_until.v proof(acotado) public_key[32] signature[64]
```

| Campo | Tipo | Restricciones |
|---|---|---|
| `name` | `str` ≤ 253 | nombre canónico **transportado en claro** (opción del propietario: la renovación pospone la muerte del TLD abierto subyacente, la cadena debe conocer el nombre); `domain_id` debe igualar `DomainId(name)` (recomputado) |
| `valid_until` | varint | nueva expiración del registro (Unix, segundos) |
| `proof` | acotado | **autorización de renovación** (ver abajo) |

Firmado por el propietario actual del dominio; que `valid_until`
extienda de verdad la expiración (y como máximo un plazo de
renovación) es una regla de estado (M8b).

#### Autorización de renovación — PoW o PoS según el TLD

El coste de una renovación depende de la naturaleza del TLD
subyacente (evaluada en aplicación, sobre el estado del bloque):

- **TLD abierto**: la renovación cuesta el **mismo PoW que la
  creación** — mismo reto (`SCONE-DOMAIN-V1 ‖ nombre canónico`),
  misma dificultad por longitud de etiqueta y por red, misma
  ventana de validez de época. Un dominio nunca se «recoge» gratis:
  cada plazo de vida se paga en trabajo.
- **TLD privativo**: el espacio de nombres es el stake del
  propietario del TLD (PoS). La transacción debe estar **firmada
  por el propietario del TLD** (o su operador delegado) — la firma
  ES la autorización, el campo `proof` debe estar vacío (20 bytes
  cero). El propietario del TLD renueva los dominios de su espacio
  bajo sus propias reglas.

### RENEW_TLD (0x8F)

```text
tld(str ≤ 63) tld_id[32] owner[32] valid_until.v proof(acotado) public_key[32] signature[64]
```

| Campo | Tipo | Restricciones |
|---|---|---|
| `tld` | `str` ≤ 63 | nombre canónico del TLD; `tld_id` debe igualar `TldId(tld)` (recomputado) |
| `valid_until` | varint | debe ser EXACTAMENTE `now + 3 años` (plazo fijo — rederivado en aplicación, cualquier otro valor rechazado) |
| `proof` | acotado | PoW de renovación (ver abajo) |

Firmado por el propietario **privativo** actual del TLD. Reglas de
aplicación:

- aplica solo a un TLD PRIVATIVO (un TLD abierto vive a través de
  sus dominios — rechazo tipado `TldIsOpen`);
- válido durante el plazo Y a lo largo de la **gracia de 60 días**
  que sigue a la expiración; más allá, el TLD está muerto (GC) y el
  rechazo es `TldExpiredFrozen`;
- **PoW igual a la creación**: reto
  `SCONE-TLD-V1 ‖ nombre ‖ expiración_actual(BE64)` (el sufijo
  anti-reutilización liga cada PoW a UN solo plazo), misma
  dificultad por longitud que la reclamación inicial;
- la renovación NO se congela durante la gracia — es la única
  operación de gestión aún disponible para un propietario
  caducado.

## UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

| Campo | Tipo | Restricciones |
|---|---|---|
| `domain_id` | 32 bytes crudos | dominio destino |
| `owner` | 32 bytes crudos | derivación de `public_key` |
| `sequence` | varint | **> 0**, debe igualar `current + 1` en aplicación |
| `record_hash` | 32 bytes crudos | compromiso sobre el conjunto de registros DHT |
| `public_key` | 32 bytes crudos | clave pública Ed25519 del firmante |
| `signature` | **exactamente 64 bytes** | firma Ed25519 del payload firmado |

Nota: `UpdateDomain` **no** lleva `timestamp` — el orden pasa por
`sequence`. Tampoco lleva el nombre: el dominio ya está registrado,
el id basta.

### Cotas estrictas

- `public_key`: **exactamente 32 bytes crudos**, sin prefijo de
  longitud. Una pk que no decodifique como punto de la curva →
  error `InvalidTransactionKey` en decodificación.
- `signature`: **exactamente 64 bytes crudos**, sin prefijo de
  longitud, sin margen. Cualquier otro tamaño es un error duro de
  decodificación (truncamiento rechazado).
- `proof`: varint de longitud + bytes, acotado a 256
  (`MAX_PROOF_LEN`), comprobado antes de asignar.
- `name`: varint de longitud + bytes UTF-8, acotado a 253
  (`MAX_NAME_LEN`), revalidado por las reglas de nomenclatura de
  `scone-core`.

## Derivación del propietario

Regla dura: **el propietario siempre se recomputa, nunca se
confía**.

```text
PublicKeyRef = BLAKE3-256("SCONE-PUBKEY-V1" || public_key)
OwnerId      = BLAKE3-256("SCONE-OWNER-V1" || PublicKeyRef)
```

Una transacción cuyo `owner` no coincida con la derivación de su
propio `public_key` es inválida (`InvalidOwner` en el lado core,
`OwnerKeyMismatch` en el lado cadena), rechazada en codificación,
decodificación **y** validación de cadena (defensa en
profundidad).

## Payload firmado

El payload firmado es la codificación canónica de la transacción
**sin el campo firma**, prefijada por una etiqueta de separación de
dominios — **con el `network_id` de la cadena destino inyectado a
la cabeza**:

```text
signing_payload(tx, network) = "SCONE-TX-SIG-V1"
                              || canonical_encode(tx_without_signature, network)
```

donde `canonical_encode(tx_without_signature, network)` sigue la
estructura wire anterior, discriminante y byte de versión
incluidos, deteniéndose antes de `signature`, **con `network` como
primer campo** — este es el ÚNICO lugar donde aparece la red. La
cadena recomputa el payload con SU propia red: una transacción
firmada para otra red falla la verificación de firma
(`InvalidSignature`); este es el mecanismo anti replay entre
cadenas.

```text
REGISTER_DOMAIN_no_sig = disc(0x21) version(0x01) network name domain_id[32] owner[32]
                            timestamp.v proof public_key[32]
REGISTER_TLD_no_sig    = disc(0x93) version(0x01) network tld tld_id[32] owner[32]
                            timestamp.v proof public_key[32]
UPDATE_DOMAIN_no_sig   = disc(0x52) version(0x01) network domain_id[32] owner[32]
                            sequence.v record_hash[32] public_key[32]
TRANSFER_TLD_no_sig    = disc(0x47) version(0x01) network tld_id[32] owner[32]
                            new_owner[32] public_key[32]
REVOKE_TLD_no_sig      = disc(0x6B) version(0x01) network tld_id[32] owner[32]
                            public_key[32]
SET_TLD_OPEN_no_sig    = disc(0xB8) version(0x01) network tld_id[32] owner[32]
                            open(0x00|0x01) open_proof(proof) public_key[32]
ASSIGN_DOMAIN_no_sig   = disc(0xD4) version(0x01) network name domain_id[32] owner[32]
                            assignee[32] public_key[32]
RENEW_DOMAIN_no_sig    = disc(0x3C) version(0x01) network name domain_id[32] owner[32]
                            valid_until.v proof public_key[32]
TRANSFER_DOMAIN_no_sig = disc(0xE2) version(0x01) network domain_id[32] owner[32]
                            new_owner[32] public_key[32]
```

Propiedades:

- determinista y puro: misma transacción ⇒ mismo payload en todas
  partes;
- el prefijo ASCII `SCONE-TX-SIG-V1` (15 bytes,
  hex `53434f4e452d54582d5349472d5631`) impide toda reutilización
  de la firma en otro dominio de hash del protocolo;
- todos los campos salvo `signature` quedan cubiertos — **incluido
  el nombre transportado por un `RegisterDomain`**: modificar
  cualquiera de ellos invalida la firma;
- el `network_id` inyectado queda cubierto: la misma tx sin firmar
  firmada para dos redes produce dos payloads distintos (y por
  tanto dos firmas distintas).

## Prueba de trabajo de registro (PoW)

El campo `proof` de `RegisterDomain` y `RegisterTld` lleva la
prueba de trabajo de registro (módulo `scone-core::pow`).

### Digest (salt de época, anti pre-mining)

```text
epoch  = block_height / POW_WINDOW        (POW_WINDOW = 144 bloques)
salt   = BLAKE3-256("SCONE-POW-V1-ARGON2" || network || challenge || epoch_le64)
digest = Argon2id(m, t, p = 1; password = nonce_le64, salt = salt[0..16])
```

`network` es el `network_id` canónico (M8b): una prueba minada
para una red no es prueba en otra.

`SCONE-POW-V1-ARGON2` fija además los parámetros Argon2id por red
(`pow_params`: mainnet m = 64 MiB, t = 2, p = 1; testnet/devnet
m = 8 KiB, t = 1, p = 1): el núcleo Argon2id (RFC 9106) hace el
coste por hash dominado por memoria (resistencia GPU/ASIC), y una
prueba minada bajo un juego de parámetros nunca verifica bajo
otro.

`challenge` es la entrada de derivación del objeto reclamado —
`"SCONE-TLD-V1" || tld` para un TLD, `"SCONE-DOMAIN-V1" || name`
para un dominio: exactamente los bytes hasheados para `TldId` /
`DomainId`, de modo que una prueba nunca es reutilizable entre
espacios de nombres ni entre clases de objeto. `epoch_le64` y
`nonce_le64` son las codificaciones little-endian de 8 bytes de la
época y del nonce.

La prueba es válida cuando `digest` tiene al menos `difficulty`
bits cero a la izquierda.

#### Salt de época — anti pre-mining

Un digest sin salt no liga ni tiempo ni altura: cualquiera podría
pre-minar hoy pruebas para miles de nombres y luego registrarlos
en ráfaga al lanzamiento de la mainnet (squatting) — la prueba era
autocontenida y válida en el bloque que llegara primero. El digest
se sala con la **época del bloque que APLICA la transacción**:

- `epoch = block_height / POW_WINDOW`, `POW_WINDOW = 144` bloques
  (~2,6 h a 65 s/bloque): una ventana de gracia para el usuario
  honesto (minar, luego enviar dentro de la ventana), re-minado
  obligatorio más allá;
- la prueba DECLARA su época en el wire; en aplicación, la época
  declarada debe caer en la **ventana de validez** del bloque
  aplicante — tolerancia SOLO hacia atrás (el envío tardío se
  absorbe, minar «para el futuro» es imposible), ventana
  dimensionada por `epoch_validity_window(difficulty)` =
  `1 << max(0, difficulty − 16)` épocas (ventana ampliada: las
  dificultades altas de mainnet se minan en días-a-semanas, más que
  una época; acaparar no gana nada porque la ventana ≈ el coste de
  minado — cada prueba de una ráfaga cuesta sus `2^difficulty`
  hashes completos DENTRO de la ventana). A dificultad ≤ 16 bits
  (testnet, nombres largos) la regla de igualdad estricta no
  cambia;
- admisión en mempool (best effort): se desconoce la altura del
  bloque futuro — el relay verifica contra la época de
  `altura actual + 1`; la verificación estricta en tiempo de
  aplicación sigue siendo la autoridad;
- replay determinista: la prueba LLEVA su época y la ventana es
  función pura de (dificultad, época del bloque) — un bloque
  histórico se reverifica idénticamente en cualquier nodo.

Cambio rupturista pre-mainnet: el formato compacto antiguo (12
bytes) se rechaza sin compatibilidad hacia atrás.

### Payload wire (20 bytes fijos)

```text
nonce_le64 || epoch_le64 || difficulty_le32
```

La dificultad y la época viajan con la prueba, pero son
**declaradas por el productor y por tanto nunca confiadas**: el
verificador compara la dificultad contra la **constante del
protocolo** para la clase de registro (un payload cuya dificultad
declarada difiera de la constante — menor *o* mayor — se rechaza,
`InvalidProof`), y la época contra la del bloque aplicante
(`ProofEpochMismatch` en desajuste). El nonce se reverifica luego
a la dificultad constante, con el digest recomputado desde cero.

### Dificultades (curva **por longitud**, M8b + calibración medida)

La dificultad NO es un parámetro plano: es función determinista de
la **longitud de la etiqueta creada** (`scone-core::pow_length` —
el TLD para `RegisterTld`, la etiqueta más a la izquierda para
`RegisterDomain`), por red:

```text
difficulty(len) = apex − round(0.415 × (len − 1)), recortado en el piso
```

| Red | TLD privativo (ápice/piso) | TLD nacido abierto / dominio (ápice/piso) |
|---|---|---|
| testnet | 8 bits / 2 bits | 6 bits / 2 bits |
| mainnet | 42 bits / 30 bits | 40 bits / 30 bits |
| devnet | 2 bits / 1 bit | 1 bit / 1 bit |

Un TLD **nacido abierto** (`RegisterTld` con `open = 0x01`) paga la
curva «de apertura» (ápice − 2, como `domain_difficulty`); **abrir
un TLD privativo existente** (`SetTldOpen`) se queda en dificultad
plana (`open_tld_pow_difficulty`: 6 bits testnet / 22 bits mainnet
/ 2 bits devnet — la tx lleva solo el `tld_id`, el nombre ya pagó
su longitud en la creación). Benchmarks de mainnet (TLD privativo,
~6 M hashes/s): 1 carácter = 42 bits (≥ 7 días), 5 caracteres =
40 bits (~2 días), 19 caracteres = 35 bits (~1,6 h), 29+
caracteres = piso de 30 bits (~3 min). Las curvas viven en
`scone-core::pow_length` (constantes `MAINNET_APEX = 42`,
`MAINNET_FLOOR = 30`, `TESTNET_APEX = 8`, `TESTNET_FLOOR = 2`,
`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1`); los parámetros planos
(`tld_pow_difficulty`, etc.) permanecen en
`scone-core::network::NetworkParams`. Cambiar un valor = un cambio
de consenso **para esa red**. Invariante: la devnet es un tercer
`NetworkParams` completo y la testnet sigue estrictamente idéntica
a sus constantes pre-devnet (parámetros Argon2id, curvas,
ventanas) — la devnet NO comparte constante alguna con las otras
redes más allá del código mismo.

Calibración medida de los topes planos (2026-10, `cargo run
--release -p scone-core --example pow_bench` — rendimiento
monohilo ~7,7 M H/s en un x86-64 de referencia):

| Clase (mainnet) | dificultad | tiempo real medio |
|---|---|---|
| `RegisterTld` | 24 bits | ~7 s |
| `RegisterDomain` | 20 bits | ~54 ms |

A ~65 s/bloque: una reclamación de TLD cuesta ~0,1 bloque de CPU
mononúcleo, un dominio ~1 ms de bloque — simbólico para un
usuario, costoso de industrializar a escala (anti-squat).
Recalibrar si los objetivos de hardware bajan (ARM móvil: ~4–8×
más lento).

Ramas de estado (M8b, entregado): `RegisterTld` SIEMPRE exige su
PoW; `RegisterDomain` lo exige sobre un TLD **abierto** (la ruta
cerrada es solo asignación — `AssignDomain`, sin PoW, firmado por
el propietario del TLD).

## Reglas de validación

Una transacción es válida si y solo si:

1. **decodificación estricta**: estructura y cotas anteriores
   respetadas (discriminante conocido, versión `0x01`, pk 32 B,
   firma 64 B, invariantes core re-checkeadas en decodificación);
2. **ligadura owner/pk**: `owner == OwnerId::from(public_key)`
   (recomputado);
3. **coherencia nombre/id** (RegisterDomain): `domain_id ==
   DomainId(name)` (recomputado);
4. **firma**: Ed25519 `verify_strict` de `signature` contra el
   payload firmado **recomputado desde cero** con la red de la
   cadena (nunca tomado de un par ni de la transacción tal como se
   recibió) — una tx firmada para otra red falla aquí
   (`InvalidSignature`, anti replay entre cadenas);
5. **reglas de estado** (en aplicación dentro de un bloque — M8b,
   ver `/docs/en/technical/blockchain.md`):
   RegisterTld → TLD libre + no ICANN + PoW (dificultad por
   longitud); RegisterDomain → TLD registrado **y abierto** +
   dominio libre (gracia incluida) + PoW (por longitud);
   UpdateDomain → dominio existente, `owner` actual,
   `sequence == current + 1`;
   TransferTld / RevokeTld / SetTldOpen / AssignDomain → TLD
   existente **y privativo**, firmante = propietario del TLD;
   TransferDomain → dominio existente, propietario;
   RenewDomain → dominio existente, extensión estricta ≤ 3 años,
   autorización PoW (TLD abierto: mismo PoW que la creación,
   **ligado a la expiración actual** — reto distinto del registro
   y de cada renovación anterior) o PoS (TLD privativo: firma del
   propietario del TLD — o de su operador efectivo —, `proof`
   vacío).

La verificación usa `verify_strict` (rechazo de firmas maleables y
claves débiles/no canónicas), como exige el dato no confiable.

## Orden de las operaciones en `push_block`

La validación criptográfica (2 a 4) tiene lugar para cada
transacción **antes** de la aplicación del estado, sobre un estado
de trabajo atómico por bloque; cualquier error deja la cadena
rigurosamente sin cambios.

## Ensamblado del bloque

`BlockBuilder` (`scone-blockchain`) ensambla un bloque a partir de
una cola de transacciones: `tx_root` recomputado sobre la lista
ordenada, cabecera encadenada al hash padre (`prev_hash`),
`version` = `PROTOCOL_VERSION`. El payload de consenso (payload
firmado del productor) y el timestamp los aporta el llamante; el
trait `Consensus` existente se usa simétricamente con la
validación.

## Ejemplos hex (vectores reales reproducibles — M8b)

### RegisterDomain `example.uip`, timestamp 42, testnet

Payload firmado (vector fijado; generado históricamente por las
herramientas offline eliminadas — los valores siguen siendo
vectores válidos). El `network` aparece aquí porque este es el
PAYLOAD DE FIRMA (su único lugar); la transacción wire firmada en
sí no contiene ninguno:

```text
53434f4e452d54582d5349472d5631       prefijo SCONE-TX-SIG-V1
21                                    disc REGISTER_DOMAIN (0x21)
01                                    versión de formato
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
0b 6578616d706c652e756970             name (str, longitud 11)
e6a2cbe264f81ed7da4d901c4deec6a3579224b8080fad01cb7461b8c264f181  domain_id
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner (clave de build fijada)
2a                                    timestamp (varint, 42)
14 0200000000000000 0000000000000000 04000000
                                      proof (20 B: nonce 2, época 0, dificultad 4 — testnet)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk + inicio de sig
```

### RegisterTld `uip`, timestamp 42

Payload firmado (vector fijado) — el `TldId` es el vector de
derivación fijado en `/docs/en/general/naming.md`:

```text
53434f4e452d54582d5349472d5631       prefijo SCONE-TX-SIG-V1
93 01                                 disc REGISTER_TLD + versión
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
03 756970                             tld (str, "uip")
ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e  tld_id ("uip")
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner
2a                                    timestamp (varint, 42)
14 8e00000000000000 0000000000000000 07000000
                                      proof (20 B: nonce 0x8e, época 0, dificultad 7 — testnet, etiqueta de 3 caracteres)
00                                    open (0x00 = nacimiento privativo)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk
```

(Los valores completos eran reproducibles vía las herramientas
offline eliminadas; los vectores siguientes siguen fijados y
válidos. El PoW se mina automáticamente a la dificultad de la red
— los nonces varían de ejecución en ejecución; solo
red/nombre/id/owner/pk son estables. Desde el cambio de wire de
2026-09, el hex de la transacción firmada YA NO contiene los bytes
`0d 73636f6e652d746573746e6574`: la red viaja solo en el payload a
firmar.)

## Dónde vive el hex ahora

Los comandos offline `tx build|sign|verify` fueron eliminados
(cli-simplify-v1). El formato hex firmado en sí no cambia y sigue
siendo el formato wire del **flujo web/API**: la aplicación web del
registrar construye, firma (keystore en el navegador) y envía
transacciones como hex canónico por la API. En el lado del CLI,
las rutas de usuario son las formas de un solo comando:

```bash
scone domain register example.uip        # mina, firma, envía
scone renew example.uip                  # prueba de renovación, firma, envía
scone domains                            # lo que posees
```

En todas las rutas de código el propietario y la clave se
**recomputan** a partir de la identidad firmante, nunca tomados de
ninguna entrada.

## Historial de versiones

| Versión | Contenido | Estado |
|---|---|---|
| 1 | `public_key` + `signature` embebidas, payload `SCONE-TX-SIG-V1`, byte de versión `0x01`, `RegisterDomain` transporta el nombre, `RegisterTld` (0x93), familia M8a: `TransferTld` (0x47), `RevokeTld` (0x6B), `SetTldOpen` (0xB8), `AssignDomain` (0xD4), `RenewDomain` (0x3C) — formatos wire normativos; M8b: campo `network` firmado a la cabeza de cada payload, `RegisterTld` transporta el nombre, reglas de estado completas, dificultades PoW por red | obsoleto |
| 1 (2026-09, wire sin network) | **eliminación del campo `network` del wire de todas las transacciones**: el network_id vive solo en el payload de firma (`signing_payload(tx, network)` lo inyecta a la cabeza); la separación de redes descansa en la firma (`InvalidSignature` sobre un payload ajeno, el error `WrongNetwork` desaparece); `TX_FORMAT_VERSION` se queda en `0x01` (testnet relanzable, sin coexistencia); **todos los TxId cambian** (el TxId es el hash de la codificación wire firmada, que pierde 14–15 bytes); digest PoW, génesis y data-dirs por red sin cambios | obsoleto |
| 1 (2026-10, salt de época PoW) | **digest PoW anti pre-mining**: salt `BLAKE3("SCONE-POW-V1-ARGON2" ‖ network ‖ challenge ‖ epoch_le)` + núcleo Argon2id sobre `nonce_le`, con `epoch = block_height / 144` del bloque que APLICA la tx; payload wire de la prueba 12 → **20 bytes** (`nonce_le ‖ epoch_le ‖ difficulty_le`), la época declarada se verifica contra la ventana de validez de época del bloque aplicante (rechazo tipado `ProofEpochMismatch`), best-effort `(height+1)/144` en la admisión al mempool, flag CLI `--epoch` en las rutas offline; cambio rupturista pre-mainnet: el payload antiguo de 12 bytes se rechaza, **todos los TxId cambian** (el campo `proof` crece 8 bytes) | actual |
