# Documentación de Scone

> Toda la documentación bajo `docs/` es **NORMATIVA**: debe permitir una
> implementación independiente sin leer el Rust. Cualquier divergencia
> documentación/código es un bug (identificar la fuente de verdad y
> corregirla).

Dos partes:

- **`general/`** — cómo funciona: los conceptos y las reglas del
  protocolo, accesibles sin leer los detalles de los formatos;
- **`technical/`** — especificaciones detalladas para desarrolladores:
  formatos binarios exactos, fórmulas de hash, límites, APIs y el
  comportamiento de cada componente.

## `general/` — cómo funciona

| Documento | Contenido |
|---|---|
| [architecture.md](general/architecture.md) | Separación de autoridad (blockchain) / disponibilidad (DHT) / almacenamiento local; roles de los crates; flujo de resolución DNS; evolutividad |
| [naming.md](general/naming.md) | Reglas de nomenclatura: alfabeto LDH, TLDs, dominios y subdominios, canonización, derivación de `DomainId` y representación hexadecimal |

## `technical/` — especificaciones para desarrolladores

| Documento | Contenido |
|---|---|
| [protocol.md](technical/protocol.md) | Formato binario canónico del wire: varints LEB128, identificadores, registros DNS, bloques, mensajes P2P, límites, versionado |
| [transactions.md](technical/transactions.md) | Formatos de transacción v1: formas firmadas (campos, orden del wire — RegisterDomain lleva el nombre, discriminantes arbitrarios, derivación del propietario, payload firmado `SCONE-TX-SIG-V1`) y las cuatro formas pow-to-owner sin firma (la prueba minada ES la autorización), reglas de validación |
| [blockchain.md](technical/blockchain.md) | Capa blockchain: `TxId`, árbol Merkle, `BlockHash`, génesis, estado canónico, validación de bloques, abstracción de consenso |
| [keystore.md](technical/keystore.md) | Formato del archivo de claves `.sconekey` (Argon2id + XChaCha20-Poly1305), API de `scone-keystore`, comandos `identity` |
| [storage.md](technical/storage.md) | Persistencia local con redb: esquema de tablas, deltas atómicos, política de memoria, modelo de confianza, límites DoS |
| [relay.md](technical/relay.md) | Relay de red (`scone-network`): protocolos libp2p, descubrimiento de pares (seeds, directorio, UPnP), alcanzabilidad (AutoNAT, circuit-relay, DCUtR), mempool, sync, DHT Kademlia, topología local, métricas Prometheus, snapshot DNS, CLI completo, configuración `.env` |
| [dns.md](technical/dns.md) | Servidor DNS UDP + TCP (M6/M7b): snapshot DNS autoritativo, caché de hot-path dividida en 16 shards (entradas compartidas, búsqueda sin allocaciones, registros empaquetados, compresión de nombres), códec RFC 1035, TCP RFC 7766 (presupuesto de 512 bytes UDP / 64 KiB TCP, bit TC), caché negativa DHT, fallback recursivo, límites |
| [cli.md](technical/cli.md) | Binario `scone`: logging y verbosidad (stdout/stderr, niveles `-v`, `RUST_LOG`), configuración por entorno vía `.env` (precedencia flag > `.env` > por defecto) |
| [security-model.md](technical/security-model.md) | Modelo de seguridad: selección del comité, anti-grinding del seed (análisis: VRF no es necesario), equivocación y slashing |
| [audit-methodologie.md](technical/audit-methodologie.md) | Metodología de auditoría de seguridad: divergencias, fuente de verdad, pasadas de corrección |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | RFC de prueba de servicio (proof-of-service): heartbeats de operadores, delegación de operaciones, elección del comité entre asientos frescos |

## Convenciones de referencia cruzada

Los documentos internos y los comentarios del código Rust hacen
referencia a estas páginas por ruta absoluta desde la raíz del
repositorio (p. ej. `/docs/en/technical/protocol.md`).
