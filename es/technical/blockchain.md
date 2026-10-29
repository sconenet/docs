# Blockchain

Documento de referencia de la capa blockchain: identidades, hashing,
génesis, estado canónico, ordenación de transacciones y validación
de bloques. La implementación de referencia es el crate
[`scone-blockchain`](../../../crates/scone-blockchain), sobre
`scone-protocol` (codificación canónica), `scone-core` (reglas de
negocio) y `scone-crypto` (BLAKE3).

Alcance actual: lógica blockchain con consenso real (productor
firmado, checkpoints, finalidad), **estado en memoria** sobre un
backend KV (el almacenamiento concreto vive en `scone-storage`) —
ver [Aplazado al consenso](#aplazado-al-consenso).

## Identidades y separación de dominios de hash

Cuatro identidades, cuatro roles, cuatro prefijos BLAKE3 distintos
— un prefijo nunca se reutiliza entre dos usos:

| Tipo | Rol | Prefijo | Fórmula |
|---|---|---|---|
| `DomainId` | identidad del nombre de dominio | `SCONE-DOMAIN-V1` | ver `/docs/en/general/naming.md` |
| `RecordHash` | identidad del contenido DNS | `SCONE-RECORD-V1` | ver `/docs/en/technical/protocol.md` |
| `TxId` | identidad de la transacción | `SCONE-TX-V1` | abajo |
| `BlockHash` | identidad del bloque | `SCONE-BLOCK-V1` | abajo |

El árbol Merkle usa su propio prefijo `SCONE-MERKLE-V1`. Estos
prefijos se añaden a los de `scone-core` (`SCONE-PUBKEY-V1`,
`SCONE-OWNER-V1`).

Cadena de dependencias completa:

```text
DomainId
    ↓
RegisterDomain / UpdateDomain
    ↓
Transaction
    ↓
TxId
    ↓
MerkleRoot
    ↓
BlockHeader
    ↓
BlockHash
    ↓
prev_hash del bloque siguiente
    ↓
cadena canónica
```

## TxId

```text
TxId = BLAKE3-256("SCONE-TX-V1" || canonical_encode(Transaction))
```

- determinista: la misma transacción (misma codificación canónica)
  ⇒ mismo `TxId`, en todos los nodos, para siempre;
- identidad **por contenido**: cualquier campo modificado cambia el
  `TxId` (dominio, propietario, timestamp, proof, secuencia,
  record_hash);
- ningún timestamp local ni valor no canónico entra en el cálculo;
- sin dependencia del almacenamiento ni de la red;
- la identidad no garantiza unicidad de aplicación: reproducir una
  transacción idéntica falla por reglas de estado (doble
  `RegisterDomain`, replay de secuencia), no por el `TxId`.

API: `scone_blockchain::transaction_id(&Transaction) -> Result<TxId>`.

## Árbol Merkle (tx_root)

Compromiso sobre la **lista ordenada** de las transacciones de un
bloque:

```text
0 transacciones  root = BLAKE3-256("SCONE-MERKLE-V1" || "EMPTY")
hoja            BLAKE3-256("SCONE-MERKLE-V1" || 0x00 || TxId)
nodo interno    BLAKE3-256("SCONE-MERKLE-V1" || 0x01 || left || right)
nivel impar     el último hash se duplica para completar el par
```

Reglas:

- **nunca se ordena**: el orden de las transacciones en un bloque
  es significativo; dos bloques con las mismas transacciones en
  orden distinto tienen raíces distintas;
- las etiquetas `0x00` / `0x01` impiden toda confusión entre una
  hoja y un nodo interno (segunda preimagen);
- el prefijo `SCONE-MERKLE-V1` aísla el árbol de los otros dominios
  de hash (un `TxId` no puede interpretarse como un nodo);
- implementación propia (~30 líneas), determinista y probada; sin
  librería Merkle externa.

API: `merkle_root(&[TxId]) -> MerkleRoot` y
`tx_root(&[Transaction]) -> Result<MerkleRoot>`.

## BlockHash

```text
BlockHash = BLAKE3-256("SCONE-BLOCK-V1" || canonical_encode(BlockHeader))
```

El hash cubre **solo** la cabecera canónica: las transacciones ya
están comprometidas por `tx_root`, así que no se hashean una segunda
vez:

```text
transacciones -> TxIds -> MerkleRoot -> BlockHeader -> BlockHash
```

Modificar una transacción, su orden, la altura, el timestamp, el
`prev_hash` o el payload de consenso cambia el `BlockHash`.

API: `block_hash(&BlockHeader) -> Result<BlockHash>`.

## Redes y génesis (M8b)

Scone define tres redes disjuntas, identificadas por un
`network_id` canónico (1..=16 bytes ASCII `[a-z0-9-]`):

| Red | `network_id` | ápice TLD privativo | ápice TLD abierto | ápice dominio | piso TLD | piso dominio |
|---|---|---|---|---|---|---|
| testnet | `scone-testnet` | 8 bits | 7 bits | 6 bits | 2 bits | 1 bit |
| mainnet | `scone-mainnet` | 28 bits | 27 bits | 26 bits | 25 bits | 22 bits |
| devnet | `scone-devnet` | 2 bits | 1 bit | 1 bit | 1 bit | 1 bit |

Los parámetros viven en `scone-core::network::NetworkParams`
(instancias con nombre `TESTNET` / `MAINNET` / `DEVNET`, búsqueda
vía `NetworkParams::by_name`). Cambiar una dificultad = un cambio
de consenso **para esa red**.

**Invariante estructural**: la devnet es un tercer `NetworkParams`
completo — nunca un subconjunto de flags de testnet. La testnet se
mantiene **estrictamente idéntica** a lo que era antes de añadir la
devnet (mismas constantes Argon2id, mismas curvas, mismas ventanas
de época); la suite de tests fija este invariante.

**Dificultad variable por longitud** (`scone-core::pow_length`):
la dificultad de una creación depende de la longitud de la
**etiqueta creada** (el TLD para `RegisterTld` — privativo o
recién abierto —, la etiqueta más a la izquierda para
`RegisterDomain`). La fórmula da:

```text
TLD     : difficulty(len) = apex − round(0.287 × (len − 1)), recortada en el piso
dominio : difficulty(len) = apex − round(0.480 × (len − 1)), recortada en el piso
```

Dos pendientes distintas: `log₂(1.22) ≈ 0.287` bit/carácter para
los TLDs (cada carácter extra divide el coste por ~1.22), y
`log₂(1.40) ≈ 0.480` bit/carácter para los dominios — los dominios
decaen más rápido: su valor está en el TLD que los lleva. Un TLD
nacido abierto se sitúa en el ápice privativo − 1; un dominio en el
ápice privativo − 2 (mainnet: TLD privativo 25, TLD abierto 24,
dominio 23). Benchmarks de mainnet (Argon2id memory-hard, ~11,4 H/s
monohilo): TLD privativo 1 carácter = 28 bits ≈ 30 días, 9+
caracteres = piso de 25 bits ≈ 3,8 días; TLD abierto 1 carácter =
27 bits ≈ 15 días; dominio 1 carácter = 26 bits ≈ 3,8 días, 9+
caracteres = piso de 22 bits ≈ 11,5 h. **Máquina de referencia de
8 núcleos** (recalibración 2026-10 — el CLI mina multinúcleo),
~11,4 H/s/núcleo. **Abrir** un TLD privativo existente
(`SetTldOpen`) se queda en dificultad plana
(`open_tld_pow_difficulty`: 6 bits testnet / 22 bits mainnet): la tx
lleva solo el `tld_id`, el nombre ya pagó su longitud en la
creación. **Dos pruebas planas más** completan la familia (antispam
a nivel de consenso, misma sal de época y ventana que los
registros): `transfer_pow_difficulty` para las cesiones
(`TransferTld`/`TransferDomain`, retos `SCONE-TLD-XFER-V1 ‖ tld_id`
/ `SCONE-DOMAIN-XFER-V1 ‖ domain_id`) y `assign_pow_difficulty`
para la creación bajo un TLD privativo (`AssignDomain`, reto
`SCONE-DOMAIN-ASSIGN-V1 ‖ name` — plana por decisión: la
exclusividad del espacio de nombres se pagó en la creación del
TLD, sin doble tasación por longitud). Valores indicativos: 16 bits
mainnet (calibración bench-pow 2026-10-23: ≈ 1,7 h por
cesión/assign en UN núcleo a los 10,6 H/s medidos, referencia
11,4 H/s; ≈ 25 min a 4 núcleos — costoso de industrializar,
trabajo ordinario para un propietario honesto), 4 bits testnet,
1 bit devnet. Función pura (determinismo).

El génesis es una constante de protocolo **por red**, reproducible
a partir de las constantes solas (sin aleatoriedad, sin reloj
local, sin red):

| Campo | Valor |
|---|---|
| `version` | `PROTOCOL_VERSION` (`1`) |
| `height` | `0` |
| `prev_hash` | 32 bytes cero (sin bloque anterior) |
| `tx_root` | raíz Merkle de la lista vacía |
| `timestamp` | `GENESIS_TIMESTAMP = 0` (puramente estructural) |
| `consensus` | los bytes canónicos del `network_id` |
| transacciones | vacío |

Variante de **génesis de asignación** (kickstarter de la mainnet,
M-next): el campo `consensus` lleva `network_id ‖ ALLOC-V1 ‖
count.v ‖ entries` (dominios/TLDs prepagados, orden canónico por
nombre, manifiesto ≤ 60 KiB, ≤ 2048 asignaciones) — este formato
ampliado es lo que justifica `MAX_CONSENSUS_LEN` = 64 KiB (ver
`scone-blockchain::genesis_alloc`).

**Distribución de la génesis de asignación**: ningún artefacto de
génesis (`genesis.bin`, manifiesto) vive jamás en el repositorio
fuente ni en las releases de código, sea cual sea la red o la fase
del proyecto — dos archivos de génesis son dos redes. La génesis de
asignación de la mainnet nace solo en el lanzamiento (el paso a la
feature `mainnet` activada por defecto): producida por
`scone-genesis` a partir de un manifiesto firmado publicado fuera
del repositorio, su hash queda fijado en el anuncio de lanzamiento,
y solo los nodos fundadores la consumen (`--genesis`, nacimiento de
la cadena); todos los demás nodos se unen a la red por sync. La
herramienta rechaza cualquier escritura dentro de un directorio de
workspace Rust (presencia de `Cargo.toml`): una génesis no debe
poder aterrizar jamás en el árbol fuente.

El `network_id` en el payload de consenso hace que los hashes del
génesis — y por tanto toda la blockchain — sean **disjuntos entre
redes**: un bloque de testnet nunca puede engancharse a una cadena
mainnet (`UnknownParent`), ningún replay es posible sin re-firmarlo
todo.

API: `genesis_of(&NetworkParams) -> Block`,
`genesis_hash_of(NetworkId) -> BlockHash`; los alias `genesis()` /
`genesis_hash()` se refieren a la testnet (el valor por defecto de
desarrollo).

### Devnet — red local reiniciable

La devnet (`scone-devnet`) es la tercera red, dedicada a la
iteración local rápida:

- **Génesis propio**: el id de red `scone-devnet` en el payload de
  consenso da un hash de génesis distinto de AMBAS testnet y
  mainnet — ningún génesis, bloque, firma o prueba PoW cruza jamás
  la frontera (el replay es imposible).
- **PoW casi nulo**: curva fijada en el rango 0..=2 bits
  (`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1` — la decreciente por
  longitud sigue aplicando): una reclamación de TLD o dominio se
  mina en milisegundos, ejerciendo exactamente la misma ruta de
  código que las otras redes.
- **Consenso PoS rápido**: mismos parámetros que la testnet (comité
  de 4, épocas de 10 s) para una finalidad rápida en una malla
  local.
- **Reinicio**: la cadena vive en su propio directorio de datos
  (`~/.scone/devnet/` por defecto). **Borrar ese directorio y
  relanzar el relay reinicia la cadena** (génesis fresco, almacén
  virgen) — sin herramientas extra. NUNCA apuntar un directorio de
  datos de devnet al directorio de otra red: un almacén no vacío
  que contenga otra cadena se rechaza (`Corrupted`).

## Estado canónico

```text
DomainState  = { owner: OwnerId, sequence: u64, record_hash: Option<RecordHash>,
                 registered_at: u64, valid_until: u64, mutations: u64 }
TldState     = enum { Private { owner: OwnerId, mutations: u64 } | Open }
               -- enum (opción del propietario): el invariante abierto/privativo es
               -- estructural; el plazo de muerte vive fuera de la entrada
               -- (tabla tld_floors + plan de muerte journaled)
`mutations` es el contador de mutaciones MONÓTONO del namespace —
la generación CAS anti-replay de sus operaciones protegidas
(`SetRecordDelegates`, `SetTldOperator`, `SetDomainOperator`,
`TransferTld`, `TransferDomain` verifican-luego-incrementan; la
renovación lo preserva, una transferencia lo incrementa). Está
comprometido en la hoja SMT del namespace — hoja dominio
`SCONE-LEAF-DOM-V4` (contador entre los campos de vida y el
compromiso de records), hoja TLD `SCONE-LEAF-TLD-V4` (byte de
variante inyectivo; contador para `Private`) — y expuesto en la
respuesta name-state. Los clientes hacen fetch-then-bind: una
transacción construida contra una generación obsoleta es un rechazo
tipado (`StaleStateBinding`) que no muta nada. Derivación completa
y racional: `transactions.md`, «El contador de mutaciones»;
análisis de seguridad: `security-model.md`.

ChainState   = NetworkParams                  (red ligada, M8b)
               + DomainId -> DomainState      (acceso directo, en memoria)
               + TldId -> TldState            (registro de TLDs, M7b)
               + DomainId -> (expired_at, OwnerId)   (ventanas de gracia, M8b)
```

* `registered_at` / `valid_until` (M8b): expiración del registro,
  evaluada contra el **timestamp del bloque padre** (determinista,
  comprometido en la cabecera — nunca el reloj local);
* `open`: un TLD privativo puede **nacer abierto** (`RegisterTld`
  con `open = 0x01` — de nadie, autoservicio) o **cerrado** en la
  reclamación (cerrado = solo asignación), luego volverse abierto
  vía `SetTldOpen` (autorregistro con PoW) — **irreversiblemente**;

- acceso directo por `DomainId` (32 bytes): nunca una `String` como
  clave, nunca un barrido completo — condición necesaria para
  apuntar a cientos de miles de millones de dominios;
- `record_hash == None` mientras no se haya aplicado ningún
  `UpdateDomain`;
- los espacios de ids de TLD y de dominio son **disjuntos por
  construcción** (prefijos de derivación distintos, ver
  `/docs/en/general/naming.md`): registrar el TLD `uip` nunca puede
  usurpar la identidad de un dominio, ni al revés.

### Reglas de aplicación

`state.apply(transaction)` es determinista y atómico (en error, el
estado queda sin cambios):

**Todas las transacciones (M8b, wire 2026-09)**: las transacciones
ya no llevan campo `network` — la separación descansa en la
**firma**: el payload firmado inyecta el `network_id` de la cadena
(`validate_transaction(tx, network_of_the_chain)`), así que una tx
firmada para otra red falla la verificación de firma
(`InvalidSignature`, ANTES de la aplicación — una tx firmada en
testnet nunca se aplica, nunca se poolea por un nodo mainnet, y
viceversa).

**REGISTER_TLD**:

- el TLD debe estar **libre** (ausente del registro de TLDs) y **no
  reservado por la ICANN** (`IcannTldReserved`: los TLDs raíz de la
  ICANN no pertenecen a nadie en Scone);
- **PoW obligatorio**: `pow::verify` sobre el reto
  `SCONE-TLD-V1 || tld` (derivado del nombre transportado por la
  tx), a la dificultad **por longitud** de la etiqueta
  (`pow_length`): nacimiento privativo → `tld_difficulty` (ápice de
  red), nacimiento abierto (`open = 0x01`) → `open_tld_difficulty`
  (ápice − 2) — el digest se recomputa desde cero
  (salt `SCONE-POW-V1-ARGON2 ‖ network ‖ challenge ‖ epoch`, núcleo
  Argon2id), la dificultad auto-declarada debe igualar la
  esperada, y la **época declarada** por la prueba debe igualar
  `height/POW_WINDOW` del bloque aplicante (salt de época contra el
  pre-mining — rechazo tipado `ProofEpochMismatch`, ver
  `transactions.md`);
- en aplicación: `open = 0x01` → `{ Open }` (de nadie —
  autoservicio, asiento PoS liberado, piso de muerte puesto en
  `now + gracia`); `open = 0x00` → `{ Private { owner } }` —
  **cerrado por defecto** (solo asignación); la apertura es
  explícita vía `SetTldOpen`.

**REGISTER (RegisterDomain)**:

- **el TLD del nombre transportado debe estar registrado Y
  abierto**: la cadena deriva `TldId(name.tld())` por sí misma
  (nunca un valor suministrado); ausente → `UnknownTld` (D1, M7c),
  presente pero cerrado → `TldClosed` (M8b: la ruta de solo
  asignación es exclusiva);
- el dominio debe estar **libre**: ausente del estado Y fuera de
  toda ventana de gracia de un registro caducado (ver el GC
  abajo);
- **PoW obligatorio**: reto `SCONE-DOMAIN-V1 || name` a la
  **dificultad por longitud de la etiqueta más a la izquierda**
  (`pow_length::domain_difficulty` — siendo el TLD abierto, este es
  el precio de entrada en autoservicio);
- en aplicación: `{ owner, sequence: 0, record_hash: None,
  registered_at: now, valid_until: now + 1 año }` — **el registro
  dura 1 año** (365 días).

**Orden intra-bloque**: significativo. `[RegisterTld, SetTldOpen,
RegisterDomain]` dentro del mismo bloque es válido; cualquier
permutación que aplique una regla antes de que exista su
precondición se rechaza como bloque entero (undo-log).

**UPDATE (UpdateDomain)**:

- el dominio debe existir (no caducado);
- `tx.owner` debe ser el propietario actual;
- `tx.sequence` debe ser **exactamente** `current_sequence + 1`
  (sin huecos, sin replay);
- en aplicación: `sequence` avanza, `record_hash` se sustituye.

**TRANSFER_TLD (0x47)**:

- el TLD debe existir y ser **privativo** (`UnknownTld` /
  `TldIsOpen` en caso contrario — un TLD abierto no pertenece a
  nadie);
- el firmante debe ser el propietario actual (`NotTldOwner` en caso
  contrario);
- en aplicación: `owner = new_owner` (la identidad del destinatario
  es opaca; la ligadura a las claves que pueden gastarlo la hacen
  las txs siguientes). Sin secuencia: la primera transferencia
  aplicada gana (orden de la cadena).

**REVOKE_TLD (0x6B)**:

- el TLD debe existir y ser **privativo** (un TLD abierto no puede
  revocarse — `TldIsOpen`: muere naturalmente en el GC), firmante =
  propietario actual;
- en aplicación: el TLD se elimina del registro y vuelve a ser
  reclamable con un `RegisterTld` fresco (PoW incluido).

**SET_TLD_OPEN (0xB8)**:

- el TLD debe existir y ser **privativo** (ya abierto → `TldIsOpen`:
  el antiguo propietario queda borrado, ya no existe clave legítima
  que firme por él); `open = 0x00` (cierre) → rechazo tipado
  `TldCloseUnsupported` ya en validación pura: **la apertura es
  irreversible** (opción del propietario);
- firmante = propietario actual (`NotTldOwner`);
- **abrir = PoW obligatorio**: `pow::verify` sobre el reto
  `SCONE-TLD-OPEN-V1 || tld_id` a la dificultad plana de la red
  `open_tld_pow_difficulty` (6 bits testnet / 22 bits mainnet),
  salt de época incluido (la prueba viaja en el campo firmado
  `open_proof` de la tx);
- en aplicación: `TldState::Open` — **el propietario queda borrado**
  (irreversible, asiento PoS liberado), un piso de muerte se pone
  en `now + gracia`.

**ASSIGN_DOMAIN (0xD4)**:

- el TLD del nombre transportado debe existir y ser **privativo**
  (`TldIsOpen` en caso contrario: un TLD abierto es solo
  autoservicio), firmante = propietario del TLD (`NotTldOwner`);
- el dominio debe estar libre (misma regla de gracia que REGISTER);
- en aplicación: `{ owner: assignee, sequence: 0, record_hash:
  None, registered_at: now, valid_until: now + 1 año }`. **Sin
  PoW**: es el propietario del espacio de nombres quien avala.

**TRANSFER_DOMAIN (0xE2, M8c)**:

- el dominio debe existir (`UnknownDomain` en caso contrario),
  firmante = propietario actual (`NotOwner`);
- en aplicación: `owner = new_owner` (el asiento PoS sigue al
  dominio en el pool del propietario, simétrico con
  `TransferTld`). Sin secuencia: la primera transferencia aplicada
  gana (orden de la cadena).

**RENEW_DOMAIN (0x3C)**:

- el dominio debe existir; **la clase del TLD se evalúa PRIMERO**
  (endurecimiento M8c):
  - **TLD privativo**: autorización **PoS** — el firmante ES el
    propietario del TLD, o el asiento del propietario del TLD
    delega en la clave pública del firmante
    (`effective_operator(tld_owner) == renew.public_key` — la
    delegación siempre se lee DESDE el propietario del TLD, nunca
    al revés). El campo `proof` debe estar vacío (proof no vacío =
    rechazar).
  - **TLD abierto**: autorización **PoW** — el firmante es el
    propietario actual del dominio (`NotOwner` en caso contrario) y
    la prueba mina el reto de renovación: misma dificultad que la
    creación (por longitud de etiqueta), **ligada a la expiración
    actual** (`challenge = DOMAIN_ID_VERSION ‖ nombre canónico ‖
    valid_until actual`) — la prueba de una renovación es distinta
    de la del registro y de cada renovación anterior
    (anti-reutilización);
- `valid_until` debe **extender estrictamente** el plazo actual
  (`RenewalNotExtending` en caso contrario);
- `valid_until ≤ now + 3 años` (`RenewalExceedsTerm` — cota
  anti-acaparamiento: **1 plazo de renovación = 1 año, máximo 3
  años por adelantado**);
- en aplicación: `valid_until = tx.valid_until`; la renovación
  pospone también el piso de muerte del TLD abierto subyacente (la
  tx lleva el nombre canónico para ese fin).

**GC determinista (M8b)**: antes de aplicar las transacciones de un
bloque, la cadena evalúa las expiraciones al **timestamp del bloque
padre**: todo dominio cuyo `valid_until ≤ now` se elimina del
estado vivo y se aparca en una **ventana de gracia de 30 días**
(`valid_until + 30 días`). Durante la gracia, solo el antiguo
propietario puede reclamar de nuevo el nombre; después, queda
libre para todos. El GC es journaled: un bloque rechazado restaura
el estado pre-GC bit a bit.

**Ciclo de vida de los TLDs (plazos fijos)**: los TLDs llevan su
propio reloj, independiente de los dominios.

- **TLD privativo**: la reclamación abre un **plazo de 3 años**
  (`TLD_TERM_SECS`). La renovación (`RenewTld`, 0x8F) paga un **PoW
  igual a la creación** y reinicia exactamente 3 años (plazo fijo
  rederivado en aplicación). A la expiración se abre una **gracia
  de 60 días** (`TLD_GRACE_SECS`): la renovación sigue posible,
  pero **toda otra operación de gestión queda congelada** —
  notablemente `AssignDomain` (rechazo tipado `TldExpiredFrozen`):
  un propietario caducado pierde los derechos de gestión hasta que
  renueve. Los dominios ya asignados debajo viven su propio
  `valid_until` — el TLD se congela, no sus hijos. Gracia
  transcurrida sin renovación: el TLD muere en el GC (asiento PoS
  liberado, reclamable). Un `TransferTld` transporta el plazo
  restante.
- **TLD abierto**: nace (o se abre) con un **piso de 1 año**
  (`OPEN_TLD_FLOOR_SECS` — anti crear-destruir instantáneo). Cada
  dominio vivo debajo retrasa el reloj de muerte (sigue la
  expiración del dominio más reciente). Sin dominios + piso
  transcurrido: muerte en el GC, reclamable.
- **TLD abierto inmortal** (solo génesis: `.scone`, `.tor` —
  nomenclatura verificable sobre direcciones onion vía CNAME —,
  `.p2p`): la asignación `OpenImmortalTld` pone el centinela
  `until = u64::MAX` — el GC nunca puede cruzarlo y el rearme
  monótono nunca puede bajarlo: el TLD nunca expira, incluso
  vaciado de todos sus dominios. Ninguna transacción en runtime
  puede crear un inmortal (`RegisterTld` conserva su piso de 1
  año); los dominios debajo viven su ciclo normal.

Las muertes de TLD comparten la cola determinista de muertes de los
dominios (`tld_deaths`), journaled y restaurable bit a bit.

**Excepción de la altura 1**: el génesis lleva un timestamp
estructuralmente nulo (`GENESIS_TIMESTAMP` = 0, convención de
protocolo — no un reloj). El tiempo de la cadena del PRIMER bloque
es pues su propio timestamp (comprometido en su cabecera,
determinista), no el de su padre: anclar las expiraciones del
bloque 1 en 0 fijaría el piso de muerte de un TLD nacido abierto
(y el `valid_until` de un dominio nacido en el bloque 1) a la época
Unix — el GC pre-apply del bloque 2 (timestamp real) eliminaría el
recurso antes de la aplicación («dropped conflicting tx from block
production: unknown TLD», regresión 60 de testnet).

**Replay**: el replay entre redes es imposible a tres niveles —
génesis disjuntos (`UnknownParent`), `network_id` inyectado en cada
payload de firma (firma inválida en la otra red — el único lugar
donde aparece la red desde el wire 2026-09), PoW separado por red
en el digest. El replay intra-red de una tx idéntica falla por
reglas de estado (doble reclamación, secuencia), nunca por el
`TxId`.

Dos nodos de la misma red que parten del mismo estado y aplican los
mismos bloques en el mismo orden producen exactamente el mismo
estado final.

## Ordenación de transacciones

El orden es una propiedad fundamental, comprometida en todos los
niveles:

```text
orden de las transacciones en el bloque
    ↓
MerkleRoot
    ↓
BlockHash
    ↓
cadena canónica
```

Dos transacciones no son intercambiables: dentro del mismo bloque,
`[REGISTER, UPDATE]` se aplica, `[UPDATE, REGISTER]` se rechaza
(dominio inexistente en el momento del `UPDATE`). El desempate
global (que ordena el mempool, que produce el bloque) pertenece al
consenso.

## Validación de bloques (push_block)

Todo se **recomputa**, nada se da por sentado — un `tx_root`, hash,
firma o campo suministrado por un par nunca se toma al valor
aparente:

1. `prev_hash` == hash del tip canónico (si no `UnknownParent` si
   el padre es desconocido, `ParentNotTip` si es conocido pero no
   es el tip: detección mínima de forks);
2. `height` == altura del tip + 1;
3. `version` == `PROTOCOL_VERSION` **exactamente** (cualquier otra
   versión — menor o mayor — se rechaza, no se reinterpreta);
4. número de transacciones ≤ `MAX_TXS_PER_BLOCK` (4096);
5. `tx_root` recomputado sobre las transacciones en orden de
   bloque;
6. hooks de consenso (`validate_header`, `validate_tx`);
6ter. **anti-replay TXID por ventana (puerto .bak)**: una
   transacción cuyo `TxId` aparece en el índice de inclusión de los
   `REPLAY_WINDOW_BLOCKS = 256` bloques más recientes se rechaza
   (`TxReplay`). El índice (TxId → altura de inclusión) es función
   pura de la cadena canónica, podado al mismo ritmo; el índice se
   reconstruye al arrancar por el almacenamiento (re-escaneo de los
   últimos `min(256, height)` bloques persistidos — M8, ver
   `/docs/en/technical/storage.md`): un nodo reiniciado rechaza los
   mismos replays que un nodo vivo. Más allá de la ventana, la
   re-inclusión sigue sujeta a reglas de estado (doble reclamación,
   replay de secuencia).
6bis. **productor firmado (M5, puerto .bak)**: el payload
   `consensus` debe decodificar como payload de productor
   (`SCONE-BLOCK-V1 ‖ pk[32] ‖ sig[64]`, acotado); la firma cubre
   el *hash de firma del productor* = el `BlockHash` de la
   cabecera **con el campo `consensus` vaciado** (el payload vive
   en ese campo; firmar el hash completo sería circular); el
   productor debe pertenecer al conjunto autorizado en el
   `timestamp` del bloque (comité elegido por checkpoint, más
   sorteos de recuperación desbloqueados; bootstrap: propietarios
   de dominios vivos — un pool vacío = producción abierta, de lo
   contrario el primer registro sería imposible). El génesis
   (altura 0, payload vacío) es estructural y nunca concierne.
   Error tipado `InvalidProducer`;
7. para cada transacción, en orden: validación **criptográfica**
   (`validate_transaction`: ligadura owner/clave recomputada +
   `verify_strict` sobre el payload firmado recomputado desde
   cero, ver `/docs/en/technical/transactions.md`), luego
   aplicación al estado canónico bajo un **journal de
   deshacer** (undo-log: una entrada por transacción aplicada,
   coste O(transacciones del bloque), no O(dominios));
8. commit atómico: todo pasa ⇒ bloque añadido, journal descartado;
   el menor error ⇒ rebobinado del journal (orden inverso) y la
   cadena rigurosamente sin cambios, bit a bit.

Ninguna función de validación hace panic; cualquier entrada
malformada u hostil produce un `BlockchainError` tipado
(`OwnerKeyMismatch`, `InvalidSignature`, …).

## Ensamblado de bloques (BlockBuilder)

`BlockBuilder` (`scone-blockchain::builder`) ensambla un bloque a
partir de una cola de transacciones validadas:

```text
BlockBuilder::after(parent_height, prev_hash)
    .with_timestamp(t) .with_consensus(payload)
    .push_tx(tx)… .build() -> Block
```

- `tx_root` **recomputado** sobre la lista ordenada de las
  transacciones en cola (nunca suministrado);
- `prev_hash`/`height` encadenados al padre (el hash del padre lo
  computa el llamante vía `block_hash`, nunca tomado de un par);
- `version` = `PROTOCOL_VERSION`;
- la cota `MAX_TXS_PER_BLOCK` se aplica en la inserción
  (`TooManyTransactions`);
- el payload `consensus` (payload firmado del productor) y el
  `timestamp` los aporta el llamante: el consenso concreto se
  conecta vía el trait [`Consensus`](#abstracción-de-consenso)
  existente, simétricamente a la validación;
- M5: `.with_producer(&SigningKey)` firma el bloque como productor
  — el payload `consensus` se convierte en el payload firmado del
  productor (sobre el hash de firma del productor definido
  arriba) y sustituye cualquier `.with_consensus` anterior.

Un bloque ensamblado pasa `push_block` sin recodificar (probado).

## Forks y elección de rama (M3c, puerto .bak)

`push_block` sigue siendo el punto de entrada lineal (padre == tip,
si no `UnknownParent`/`ParentNotTip`). `try_attach` (M3c) evalúa
bloques competidores:

- padre desconocido → `UnknownParent` (el bloque espera);
- linaje válido pero rezagado → evaluación de rama: la cadena más
  larga gana; empate perfecto → desempate determinista (el hash de
  tip más pequeño en orden lexicográfico — dos nodos con los mismos
  bloques eligen siempre el mismo ganador, probado);
- **piso de finalidad**: una rama uno de cuyos antepasados
  contradice un checkpoint finalizado (incluido en el
  `prev_checkpoint_hash` de un checkpoint aceptado) se rechaza
  (error tipado
  `Consensus("branch contradicts a finalized checkpoint")`);
- adopción = replay desde el génesis de los bloques de la rama
  ganadora (determinismo bit-exacto probado: dos nodos que reciben
  los mismos bloques en órdenes distintos acaban en el mismo
  estado);
- ningún bloque recibido es canónico antes de la validación
  completa.

## Ventana RAM acotada (M7a, puerto .bak)

La cadena solo conserva los **últimos 512 bloques** en RAM
(`RAM_WINDOW_BLOCKS`, puerto del `HISTORY_KEEP = 512` del .bak),
por encima del **piso de finalidad** (el último checkpoint
finalizado):

- un bloque sale de RAM solo si está **ambas** cosas fuera de la
  ventana (`height < tip − 511`) **y** estrictamente por debajo del
  piso de finalidad — un reorg por encima del piso sigue siendo
  siempre posible, así que esos bloques NUNCA se desalojan;
- antes del primer checkpoint finalizado (bootstrap), **nada se
  desaloja**: la memoria acotada empieza con la finalidad;
- el desalojo es real: bloques, el índice de hashes
  (`known_hashes`, también acotado, indexado por altura) y txs
  salen de memoria — el almacén (`NodeStore`, `scone-storage`)
  guarda las copias duraderas;
- `block(h)` por debajo de la ventana → `None`; `block_result(h)`
  → error tipado `BlockPruned { height }` — las rutas de reorg
  (`push_block`, `try_attach`, `adopt_branch`, replay de prefijo)
  lo propagan: el relay debe recargar el segmento desde el
  almacenamiento o caer a un sync completo (limitación
  documentada: sin adopción por segmentos antes del puerto completo
  del relay);
- un bloque construido sobre un padre desalojado clasifica como
  `BlockPruned` (no `UnknownParent`): altura del padre <
  `base_height` ⇒ el almacén puede servirlo.

## Abstracción de consenso

```rust
pub trait Consensus {
    fn validate_header(&self, header: &BlockHeader) -> Result<()>;
    fn validate_tx(&self, tx: &Transaction) -> Result<()>;
}
```

- `PermissiveConsensus` (por defecto): lo acepta todo — se usa
  hasta que se definan las reglas reales;
- los hooks los llama `push_block`: el consenso concreto (PoW,
  dificultad, tasas, timestamp/anti-replay, elección de rama) se
  conectará sin reescribir la cadena;
- una sola abstracción, sin jerarquía de traits.

## Estado escalable (backend KV, pivote SMT, cola de expiración, pool de propietarios acotado)

Objetivo: 250+ mil millones de dominios potenciales — NUNCA cargar
todo en RAM. Cuatro piezas:

### Backend KV (`state_backend.rs`)

```rust
pub trait StateBackend: Default + Send {
    fn get(&self, key: &Pkey) -> Option<Vec<u8>>;
    fn put(&mut self, key: Pkey, value: Vec<u8>);
    fn delete(&mut self, key: &Pkey);
    fn range(&self, from: Option<&Pkey>, limit: usize)
        -> (Vec<(Pkey, Vec<u8>)>, Option<Pkey>);
}
```

- `Pkey` = los 32 bytes crudos del `DomainId`/`TldId` — UN solo
  espacio de claves ordenado (prefijos de derivación disjuntos),
  valores = las codificaciones `SCONE-ENTRY-DOM-V1` (88 bytes fijos)
  / `SCONE-ENTRY-TLD-V1` (variable: 33 bytes privativo / 1 byte
  abierto, enum de opción del propietario — flag + owner
  condicional, sin campo `until` ya);
- `MemoryBackend` (BTreeMap) = implementación de referencia (dev,
  tests); SIN dependencia concreta en `scone-blockchain` (redb
  prohibido en el crate) — protocolo/consenso dependen solo del
  TRAIT;
- `CachedBackend<B>` (`state_cache.rs`) = caché LRU acotada sobre
  un backend interno: write-through (el interno siempre está al
  día), los misses nunca se cachean, `range` pagina el interno —
  mismas respuestas que el backend interno solo (contrato de
  determinismo, probado);
- `RedbBackend` (`scone-storage::redb_backend`) = backend
  persistido: tabla `state_kv` (claves crudas de 32 bytes, orden
  lexicográfico = orden ascendente requerido), contador mantenido
  en la tabla meta `state_kv_meta` (nunca COUNT);
  `CachedBackend<RedbBackend>` = estado con huella RAM acotada — el
  «nivel ligero»;
- `smt_persist`: consume los conjuntos sucios del SMT
  (`take_dirty_sets`) y los escribe en un backend bajo claves de
  prefijo reservado (0x01 nodo, 0x02 hoja) — fundación del SMT
  paginado;
- el trait ya no exige `Default` (una base de datos en disco no
  tiene un valor por defecto sensato);
- `range` itera en orden ascendente de clave (contrato de
  determinismo) — solo ruta de archivo, nunca una regla de
  consenso;
- `ChainState` accede a dominios/TLDs SOLO por clave a través del
  backend (lecturas puntuales, escrituras journaled).

### Raíz SMT canónica (pivote)

- `state_root_smt()` (O(1), raíz cacheada, `SCONE-STATE-SMT-V1`)
  es el state_root **canónico**: es lo que compromete
  `checkpoint_data` y lo que verifica `accept_checkpoint` en el
  tip. El cálculo NUNCA recorre el conjunto de dominios;
- el fold directo antiguo `SCONE-STATE-V1` sobrevive como
  `state_root_archive()`: O(N), ruta de archivo/verificación
  (re-fold completo desde almacenamiento para contrastar una raíz
  histórica) — no es un artefacto de consenso. Ambos comprometen
  las mismas hojas: estados lógicamente iguales ⇒ raíces iguales
  (para ambos);
- un checkpoint aceptado con la raíz SMT se reverifica tras un
  restore (recarga): la raíz es función pura del estado restaurado
  (backend + SMT + índice).

### Cola de expiración (GC O(caducados), no O(N))

- `expirations: BTreeMap<u64, BTreeSet<DomainId>>` — instante de
  expiración → ids que caducan entonces (ids ordenados por slot:
  la igualdad de estado y el orden de drenaje son independientes
  del orden de llegada);
- `gc_expired_journaled(now)` drena `range(..=now)` en lugar de
  barrer todo el estado: coste O(caducados en ese instante);
- cota RAM: una entrada por dominio VIVO (solo expiraciones
  pendientes — una renovación re-clava el slot, el GC lo drena),
  nunca un nodo por dominio en una estructura de barrido;
- mantenida por el journal de deshacer (rollback bit-exacto: el
  slot restaurado se vuelve a poner).

### Pool de propietarios acotado (claves, no dominios)

- `owner_refcounts: HashMap<OwnerId, u64>` — un contador por
  propietario vivo DISTINTO (dominios + TLDs), mantenido por el
  journal en cada register/transfer/revoke/assign/GC
  (acquire/release);
- `eligible_validators(now)` lee este índice — O(propietarios),
  NUNCA una iteración sobre dominios. La vida temporal ya está
  garantizada: el GC determinista (timestamp del padre) ha
  eliminado todo dominio caducado Y su refcount antes de la
  llamada. Salida ordenada (determinista);
- cota RAM: O(propietarios distintos), independiente del número de
  dominios;
- `owner_pool_root` (fold de archivo) pliega estos propietarios
  ordenados — tampoco barrido de dominios.

Presupuesto RAM en régimen (backend disco/red):
O(propietarios distintos + expiraciones pendientes + ventanas de
gracia en vuelo + journal de deshacer) — todo acotado con
independencia del número de dominios, todo restaurado bit-exacto
por el journal.

## Asientos y el comité de finalidad (mecanismo PoS exacto)

Esta sección describe el mecanismo EXACTO de elegibilidad y
elección de comité tal como está implementado (`finality.rs`) — el
complemento operativo (bucle anchor, wire) vive en
`/docs/en/technical/relay.md`, el análisis de seguridad en
`/docs/en/technical/security-model.md`.

### Un asiento = posesión (sin registro)

Un **asiento** PoS NO es un registro: es la posesión de un dominio
o TLD vivo. `eligible_validators(now)` computa el pool:

- claves de los propietarios vivos vía el índice acotado
  `owner_refcounts` (un propietario DISTINTO por dominio/TLD vivo —
  cien dominios bajo una clave = UN asiento: el anti-Sybil
  económico es por identidad, lineal);
- un TLD cuenta (su reclamación costó un PoW — el propietario de un
  TLD vivo es un stakeholder), un dominio vivo también
  (`valid_until > now` — garantizado por el GC determinista, ya
  drenado antes de la llamada);
- **menos las claves expulsadas** (M9: un equivocador slashed está
  fuera del pool de por vida);
- un TLD **abierto** (`owner: None`) no ostenta asiento: su antiguo
  propietario se libera al abrirse (`owner_release` en el apply del
  `SetTldOpen`), y muere en el GC sin volver a votar.

Salida ordenada (orden determinista, independiente del orden de
iteración del índice).

Consecuencia estructural: **la muerte del asiento = la muerte del
dominio/TLD**. Expiración + GC, `RevokeTld` (asiento liberado),
`TransferTld`/`TransferDomain` (el asiento sigue al dominio hacia
el nuevo propietario), apertura `SetTldOpen` (asiento liberado) —
cada transición de propiedad mueve o libera el asiento en el pool,
simétricamente con el journal de `owner_refcounts`
(acquire/release).

### Pool CONGELADO en el último checkpoint finalizado (`FinalizedBase.pool`)

Al aceptar un checkpoint (`accept_checkpoint`), la base de
finalidad captura el pool del estado EN EL BLOQUE FINALIZADO:

```text
FinalizedBase { checkpoint, pool, block_ts, seed }
```

- `pool` = `eligible_validators(timestamp del bloque finalizado)`
  — la finalidad NUNCA relee después el estado vivo: cualquier
  registro (reclamación, transferencia, expulsión) posterior al
  checkpoint solo cuenta para épocas posteriores;
- `seed` = `next_seed(signing_hash(checkpoint))` — la semilla de la
  época SIGUIENTE deriva del hash del **contenido** del checkpoint
  (keyed, juego de firmas excluido — dos vistas con subconjuntos de
  quórum distintos del mismo checkpoint eligen el mismo comité).

### Sorteo del comité (bootstrap, frescura, degradación)

`committee(recovery)`:

1. **Bootstrap** (sin checkpoint finalizado): firman los
   **operadores FRESCOS** del pool (`fresh_operators` — un
   heartbeat reciente, la misma ventana que el régimen finalizado).
   El filtro es estado de la cadena (los heartbeats están
   on-chain), así que es determinista por tip: dos nodos en el
   mismo tip computan el mismo conjunto. Por debajo de
   `MIN_FINALITY_COMMITTEE_SIZE` operadores frescos, degradación
   segura al pool ENTERO (bootstrap en frío, caída general de
   operadores — la finalidad nunca se pierde por construcción). El
   pool bruto no puede servir como conjunto de firma: cada
   registro en autoservicio aumenta el denominador del quórum
   mientras esos propietarios nunca firman — la finalidad se
   volvió inalcanzable tras ~10 registros (testnet, 2026-10-21).
2. **Régimen finalizado**: sorteo desde el subconjunto **fresco**
   del pool congelado (RFC proof-of-service §5.2) — cada asiento
   (clave del propietario) se resuelve a su operador EFECTIVO por
   el estado en el tip (delegación
   `SetTldOperator`/`SetDomainOperator` = un hecho on-chain
   reciente, o la clave del propietario cuando se auto-opera); el
   asiento participa BAJO SU CLAVE EFECTIVA (esa es la que firma),
   siempre que tenga un **heartbeat fresco** (ventana
   deslizante). Las claves expulsadas siguen excluidas.
3. **Degradación segura**: menos de `MIN_FINALITY_COMMITTEE_SIZE`
   (4) operadores frescos → sorteo desde el pool ENTERO (bootstrap
   en frío, caída general de operadores — la finalidad nunca se
   pierde por construcción).

`allowed_producers(block_ts)` (producción de bloques) sigue una
regla deliberadamente más ancha: pool entero en bootstrap; **pool
CONGELADO entero** en el régimen finalizado. El comité elegido (+
sorteos de recuperación) sigue siendo solo el conjunto de FIRMA de
checkpoints. Restringir la producción al comité se especificó e
implementó primero, y luego se retiró: un productor cuya vista va
un checkpoint por detrás evalúa los candidatos con las reglas de
bootstrap y emite bloques que todo par finalizado rechaza
(`InvalidProducer`), y cualquier historial de cadena ensamblado a
través de la transición no puede pasar nunca el replay
`pool_at_height` — la finalidad se congela en el primer checkpoint
para siempre (reproductor sim `duplicate_and_delay`,
2026-10-21). La congelación del pool sola carga el peso de la
seguridad: los cambios de membresía (registro, transferencia,
expulsión, expiración) solo surten efecto en el SIGUIENTE
checkpoint, así que el conjunto de productores sigue siendo una
función pura de la cadena finalizada.

El quórum del checkpoint es `quorum_for(tamaño del comité)`
(floor(2n/3)+1); un comité por debajo de
`MIN_FINALITY_COMMITTEE_SIZE` nunca finaliza (guarda BFT).

## Aplazado al consenso

Deliberadamente indefinido en este crate:

- **mempool**: la admisión/reordenación fina (tasas, reemplazo)
  queda en el relay (M4 trajo cap por dominio + anti-replay TXID
  por ventana — `REPLAY_WINDOW_BLOCKS = 256`);
- **ordenación global**: el orden del mempool es un orden local
  determinista (relay), no una autoridad global;
- **autoridad de timestamp** más allá del GC: las expiraciones
  (M8b) usan el timestamp del bloque padre; otras reglas
  temporales (anti-replay fino) quedan en el consenso;
- almacenamiento distribuido: el estado está en memoria; el backend
  (`scone-storage`, redb) aplica estas reglas sin duplicarlas.

Entregado desde M8b (antes aplazado): PoW de registro por red
(`RegisterTld` siempre, `RegisterDomain` sobre un TLD abierto),
expiración/renovación 1 año + gracia de 30 días + horizonte de 3
años, GC determinista al timestamp del bloque padre, replay entre
redes imposible (génesis + campo firmado + digest PoW).

## Simulación distribuida determinista (P1.6)

`crates/scone-blockchain/src/sim/` contiene un arnés de simulación
de red **determinista** (`SimNet`): N `Blockchain`s reales (sin
mocks) dirigidas por un **reloj virtual** y una **cola de
eventos** priorizada por `(tick, orden de inserción)`. Sin sockets,
sin async, sin reloj de pared — cada sorteo (pérdida, latencia,
duplicación) es función pura de `(semilla, número de envío), de
modo que un escenario con semilla fija es reproducible byte a byte.

### Modelo

- **Transporte**: toda emisión sufre pérdida probabilística
  (configurable en ‰), latencia artificial acotada
  (`latency_ticks`), posible duplicación (una copia retrasada) y el
  mapa de particiones actual (los mensajes inter-grupo se pierden).
  Las duplicaciones nunca hacen avanzar el reloj lógico ni
  disparan nuevos envíos: el flujo real de eventos es idéntico con
  o sin duplicación (contrato de determinismo, demostrado por el
  test `duplicate_and_delay`).
- **Gossip epidémico**: un nodo retransmite a sus pares conectados
  (misma partición) un bloque/transacción/checkpoint solo en
  primera vista — regla de relay H2 (corta los bucles). Un bloque
  aparcado (huérfano) NUNCA se re-difunde; los huecos de recepción
  los rellena la ruta de sincronización (`GetBlocks`), igual que el
  relay.
- **Producción**: un paso global de producción por intervalo; cada
  grupo de partición produce como mucho UN bloque por ronda, por su
  nodo autorizado con la mejor altura. Durante el bootstrap (antes
  de la primera finalidad) el productor se fija al nodo más bajo:
  el conjunto de productores autorizados deriva del estado (pool de
  dominios vivos), dos vistas ligeramente distintas eligen comités
  distintos y un bloque localmente legítimo sería `InvalidProducer`
  en otra parte — una divergencia permanente. Tras la primera
  finalidad el comité queda congelado en la base de finalidad
  (idéntico en cualquier nodo convergido) y la producción es
  libre.
- **Finalidad**: en cada múltiplo de `sign_interval` alturas, el
  anchor designado del comité firma el contenido determinista de su
  tip vía un `SignerGuard` real a prueba de caídas (una clave = un
  voto por época, estado fsync-eado antes de la difusión); los
  demás miembros del comité firman al recibir (tras verificar que
  el bloque referenciado es efectivamente su bloque canónico a esa
  altura — guarda a nivel de relay) y re-difunden el agregado
  enriquecido. Al quórum, `accept_checkpoint` finaliza. Cada nodo
  se compromete con la primera propuesta vista por época (guarda
  bootstrap anti doble finalidad: dos propuestas competidoras de la
  misma época con firmantes disjuntos no pueden ambas acumular).
- **Caída/reinicio**: un nodo caído pierde todo su estado RAM
  (cadena, mempool, parking, vistas) pero conserva su directorio
  signer-guard en disco, exactamente como un proceso real; se
  resincroniza reproduciendo los bloques canónicos servidos por sus
  pares (`GetBlocks`/`GetCheckpoints`).

### Escenarios cubiertos (`sim::tests`)

| Test | Qué se demuestra |
|---|---|
| `convergence_simple` | 8 nodos, 5 ‰ de pérdida, latencia 1–5 ticks, 500 bloques: mismo tip, mismo `state_root_smt` en todas partes; finalidad activa; nunca dos checkpoints incompatibles por época. |
| `partition_then_heal` | Partición 4a/4b durante ~300 bloques: ambos lados producen, la curación → convergencia con reorg de la rama perdedora; las ventanas de checkpoint cruzadas nunca contradicen una época compartida (antes y después de la curación). |
| `crash_restart` | Caída a h≈200, reinicio a h≈400: resincronización reproduciendo los bloques de la red — misma altura, tip, estado; la tx post-caída es visible para el nodo resucitado. |
| `duplicate_and_delay` | 10 ‰ de duplicación: altura final, raíz de estado y número de checkpoints finalizados idénticos a la ejecución sin duplicación (idempotencia: TxReplay/seen-sets absorben duplicados). |
| `concurrent_tx` | Dos propietarios concurrentes sobre el mismo dominio: exactamente una actualización gana (el propietario — `NotOwner` para el intruso, rechazo tipado limpio), nunca una doble aplicación de estado, la tx perdedora no entra en ninguna cadena. |

### Parámetros y cotas de duración

Valores por defecto: `produce_interval` 7 ticks, `sign_interval` 16
alturas, `sync_interval` 29–60 ticks (41 por defecto, rebajado en
algunos tests), `horizon` 200k–500k ticks virtuales. Cada test
embebe una guarda de reloj de pared < 60 s en modo debug
(`assert_under_60s`). El coste dominante es el núcleo de validación
(~9 ms por `push_block_with_gc` vacío en debug, Ed25519 sin
optimizar): los escenarios se quedan por debajo de ~8 nodos × 500
bloques.

### Límites

- **Sin capa de transporte real**: este arnés ejercita la lógica
  de cadena bajo entregas adversas, no el swarm (libp2p,
  request-response, backpressure, descubrimiento). Las pruebas e2e
  reales (`crates/scone-network/tests/`) siguen siendo la prueba de
  transporte.
- **Sincronización por lotes en RAM**: los bloques servidos vienen
  de la ventana RAM (los escenarios se quedan bajo
  `RAM_WINDOW_BLOCKS`); la ruta `BlockPruned` + recarga del almacén
  no se simula (cubierta por los tests unitarios de la cadena).
- **Modelo de anchor honesto**: los anchors solo firman contenido
  verificable contra su propia cadena canónica; la equivocación
  bizantina (doble firma deliberada) la impide el `SignerGuard`
  pero no se ejercita como ataque activo.
- **Una partición a la vez**: el mapa de grupos es global y lo
  cambia secuencialmente el driver; las particiones continuamente
  solapadas/evolutivas no se modelan.
