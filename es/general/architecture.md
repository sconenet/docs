# Arquitectura

## Primero, en palabras sencillas

Cinco conceptos bastan para entender todo lo demás:

1. **Un nombre** — una dirección en internet, como `mysite.scone`.
2. **Tu frase de 24 palabras** — tu identidad. Es la escritura que
   demuestra que un nombre es tuyo, y firma cada cambio que haces.
3. **El libro de registros público** (la blockchain) — el libro mayor
   compartido y público de quién posee qué. Nadie lo controla, nadie
   puede reescribirlo.
4. **Minar** — cómo reservas un nombre: tu ordenador hace un
   trabajo, justo e igual para todos. Sin cola, sin intermediarios.
5. **Un relay** — un ordenador que hace funcionar la red manteniendo
   una copia del libro y respondiendo preguntas. Cualquiera puede
   ejecutar uno; cuantos más haya, más fuerte será la red.

El resto de esta página profundiza más — puedes parar aquí y ya
entender el proyecto.

## Principio rector

Scone separa la autoridad de la disponibilidad:

- la **blockchain** es la fuente de verdad sobre la propiedad y el
  orden de los cambios;
- el **DHT** es un sistema de distribución/replicación, **no** es una
  autoridad;
- el **almacenamiento local** guarda el estado del nodo, los índices
  y las cachés.

Un nodo debe tratar cualquier dato proveniente del DHT como
potencialmente malicioso: solo se acepta tras la verificación
criptográfica (firma del propietario, coincidencia con la propiedad /
secuencia / record_hash de la cadena).

## Separación de responsabilidades

```text
BLOCKCHAIN
──────────
Propiedad
Registro
Autorización
Ordenación
Secuencia
Hash de registro
Información de consenso

DHT
───
Registros DNS
Registros firmados
Replicación
Disponibilidad

ALMACENAMIENTO LOCAL
─────────────────────
Estado local
Índices
Registros DHT en caché
Estado de la blockchain
```

## Crates y dependencias

```text
scone-crypto     ← capa más baja: primitivas puras sobre bytes
    ▲
scone-core       ← tipos del protocolo (depende de scone-crypto)
    ▲
scone-protocol  ← formato binario wire/blockchain
    ▲
scone-blockchain← lógica blockchain, estado canónico (en memoria)
    ▲               └ scone-topology (motor puro, cero red/fs/
                     async; alimentado por scone-network) — scone-genesis
                     (binario de herramienta one-shot para el génesis
                     de la mainnet)
scone-storage   ← persistencia local: backend redb + snapshots verificados
    ▲
scone-network   ← relay p2p: libp2p/QUIC, mempool, sync, DHT, DNS
    ▲
scone-keystore  ← archivos de claves `.sconekey` cifrados (depende de scone-crypto)
    ▲
scone           ← binario / aplicación
```

### Decisión: `scone-core` depende de `scone-crypto`

El `DomainId` debe poder computarse desde la base
(`BLAKE3-256(prefix || canonical name)`), pero la criptografía
concreta no debe contaminar el core. Resolución:

- `scone-crypto` es la capa más baja: funciones puramente
  deterministas sobre bytes (más el RNG del SO para la generación de
  claves), cero tipos del core, sin dependencias de red/sistema de
  archivos/async → imposible la circularidad;
- `scone-core` lo usa únicamente para hashear;
- Las firmas y claves Ed25519 viven en `scone-crypto` y operan sobre
  bytes (payloads canónicos producidos por `scone-protocol`).

### Rol de cada crate

**scone-core** — nombres (`TldName`, `DomainName`), identificadores
(`DomainId`, `OwnerId`, `PublicKeyRef`), datos DNS (`DnsRecord`,
`SignedDnsRecord`), transacciones (`RegisterDomain`, `UpdateDomain`).
Determinista y portable. Prohibido en este crate: libp2p, redb,
tokio/async, red, archivos, SO, servidor DNS, lógica blockchain.

**scone-crypto** — primitivas criptográficas. Hoy: `hash256`
(BLAKE3), firmas y claves Ed25519 (`SigningKey`,
`PublicKey`, `Signature`, verificación estricta por lotes), sobre
AEAD de grupos de registros (XChaCha20-Poly1305). Previsto: árbol
Merkle. Regla: nada de algoritmos caseros, solo dependencias auditadas.

**scone-protocol** — formato binario del protocolo: codificación/
decodificación canónica (varint LEB128 mínimo, identificadores de
32 bytes puros) de transacciones, registros DNS, bloques
(`Block`/`BlockHeader` con un campo de consenso opaco) y mensajes
P2P, con límites de tamaño comprobados antes de asignar, rechazo de
formas no canónicas y versionado (`PROTOCOL_VERSION`). Sin lógica de
consenso, red ni almacenamiento.
Referencia de formato: `/docs/en/technical/protocol.md`.

**scone-blockchain** — lógica blockchain y estado canónico:
identificación de transacciones (`TxId`), árbol Merkle ordenado,
`BlockHash`, génesis determinista, validación de bloques y
transacciones, reglas de aplicación del estado
(`DomainState`/`ChainState`, en memoria, acceso directo por
`DomainId`), detección de forks y elección de rama (`fork_choice`:
piso de finalidad + cadena más larga + desempate por menor hash),
finalidad por checkpoint, y abstracción de consenso
(`trait Consensus`, implementación permisiva por defecto). El
PoW de registro (minar/verificar, dificultades por red) vive en
`scone-core`. Prohibido en este crate: red, DHT, almacenamiento
(redb), servidor DNS. Referencia: `/docs/en/technical/blockchain.md`.

**scone-storage** — persistencia local (entregada). El backend
concreto es `redb` (`NodeStore`), oculto tras un trait mínimo para
que blockchain/DHT/DNS nunca se acoplen directamente a redb; otro
backend (KV distribuido) podría conectarse. Incluye el backend de
estado KV paginado, el log de bloques, los snapshots verificados
(manifiesto hasheado + páginas) y la ruta de restauración/replay al
reiniciar. Referencia: `/docs/en/technical/storage.md`.

**scone-network** — el relay p2p del nodo: swarm libp2p/QUIC, relay
de transacciones y bloques (validación local sistemática), mempool
acotado, sync de bloques acotado (128 bloques/petición), DHT Kademlia
para `SignedDnsRecord`, servidor DNS UDP+TCP (GeoDNS siempre
activado — tabla GeoIP embebida, fallback recursivo), el bucle de
checkpoint/heartbeat de los anchors (el CLI habla p2p QUIC directo
al relay, ya sin RPC local). Alcanzabilidad y descubrimiento:
AutoNAT v2 (veredicto alcanzable/NATeado), circuit-relay v2 + DCUtR
para nodos NATeados y UPnP (mapeo del puerto QUIC por
el router, opt-in). Configuración vía un archivo `.env` (ver
`.env.example`). Referencia: `/docs/en/technical/relay.md`.

**scone-keystore** — almacenamiento cifrado de las claves de firma
Ed25519 (archivos `.sconekey`: Argon2id + XChaCha20-Poly1305). Este
crate posee el sistema de archivos para que `scone-crypto` pueda
mantenerse puro. Referencia: `/docs/en/technical/keystore.md`.

**scone-topology** — motor PURO de optimización de topología p2p
local: cero red, cero sistema de archivos, cero async. No conoce ni
libp2p ni el swarm: `scone-network` lo alimenta con observaciones
(RTT de ping, conexiones) y recibe decisiones de gestión del
vecindario.

**scone-genesis** — binario de herramienta one-shot (kickstarter)
para la preparación del génesis de la mainnet: manifiesto JSON
versionado, asignaciones, bloque determinista. La herramienta
produce los artefactos, el repositorio nunca los versiona (rechaza
escrituras dentro de un workspace Rust; distribución reservada al
lanzamiento, ver `blockchain.md`).

**scone** — binario/CLI. Depende de un crate solo cuando de verdad
lo usa (hoy: scone-network para el relay, scone-blockchain,
scone-storage, scone-keystore, scone-protocol, scone-core,
scone-crypto, scone-topology).

## El nodo

Un nodo Scone combina:

1. una blockchain (validación de consenso, estado de propiedad);
2. enrutado DHT (publicación/recuperación de `SignedDnsRecord`);
3. almacenamiento local (estado, índices, cachés);
4. un resolvedor / servidor DNS al frente.

### Arrancar un nodo nuevo (bootstrap de estado, P0.3/P0.4)

Un nodo nuevo NO reprocesa el génesis: parte de un estado
finalizado verificado.

```text
checkpoint finalizado (comité, quórum de firmas — validado por
        │              la capa de cadena antes de cualquier uso)
        ▼
snapshot verificado (manifiesto + páginas paginadas; el importador
        │              RECOMPUTA la raíz SMT a partir de las
        │              entradas y la compara con el state_root
        │              firmado — un snapshot falsificado/incompleto
        │              se rechaza antes de cualquier escritura)
        ▼
bloque ancla H (pedido a la red POR HASH)
        │
        ▼
bloques recientes H+1..H+k (relay, altura creciente)
        │
        ▼
sincronizado — al arrancar, replay de SOLO el sufijo por encima de H
```

Dos perfiles de nodo:

- **nodo archivo** (flag del operador): conserva todos los bloques
  históricos, los sirve a la red, arranca por las rutas existentes;
- **nodo de estado completo**: posee el estado completo en H y el
  sufijo, pero no el historial por debajo de H — valida los bloques
  futuros y sirve estado, no sirve historial.

Esta ruta es OPT-IN: importar un snapshot verificado es una acción
explícita del operador (`import_verified_snapshot`), nunca
automática — el comportamiento por defecto de un nodo existente no
cambia. Detalles normativos: `/docs/en/technical/storage.md`,
secciones «VERIFIED state snapshot» y «State bootstrap».

## Flujo de resolución DNS

```text
Consulta DNS (example.uip)
    │
    ▼
Nodo: DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || "example.uip")
    │
    ├── snapshot DNS autoritativo (RAM) ─► desconocido/expirado → NXDOMAIN
    │                                          (sin tocar el core)
    ├── caché DNS por shards (qname,qtype) ─► hit: entrada compartida,
    │                                          verificación perezosa
    ├── caché local (SignedDnsRecord) ─────► verificación (firma, propietario,
    │                                          secuencia, record_hash vs cadena)
    ├── DHT get(DomainId) ─────────────────► misma verificación
    │                                          (miss → caché negativa de 30 s)
    ▼
Respuesta DNS (si y solo si es válido)
```

El snapshot DNS (existencia + expiración de cada dominio vivo,
publicado tras cada bloque aplicado) garantiza que una decisión de
existencia nunca pasa por el run loop del relay: el core solo se
toca para las recuperaciones reales de registros.

## Evolutividad

- preimages versionados (`SCONE-DOMAIN-V1`, `SCONE-PUBKEY-V1`,
  `SCONE-OWNER-V1`, …): cambiar una regla de derivación crea un
  nuevo espacio de nombres en lugar de romper el existente;
- `RecordData` extensible vía variantes + `Unknown { type_code, data }`
  para tipos desconocidos en el momento de la lectura;
- `PROTOCOL_VERSION` (`scone-protocol`) para rupturas del formato
  wire;
- la cadena solo guarda hashes: los datos voluminosos quedan en el
  DHT.
