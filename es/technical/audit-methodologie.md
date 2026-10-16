# Metodología de auditoría de seguridad — Scone (pre-mainnet)

Documento de trabajo. Objetivo: confianza para el despliegue en
producción. Referencias entre crates: las reglas duras de AGENTS.md
(nada de cripto casero, decodificación estricta, hashes recomputados,
determinismo) son la especificación de requisitos de la auditoría.

## 0. Alcance y amenazas (antes de releer una sola línea de código)

Modelo de adversario: un nodo p2p arbitrariamente hostil (puede
enviar cualquier byte por QUIC/UDP), un enviador de transacciones
hostil, un operador de relay curioso, un data-dir robado. EXCLUIDO
por diseño: el cliente final (el DNS es público por naturaleza), la
disponibilidad global (el caos ya está probado).

Tabla de activos, por orden de criticidad:
1. **Propiedad** (cadena de estado: Register/Update/Transfer/Assign)
   — un falso positivo = robo de dominio.
2. **Posesión de firma/clave** (keystore, Ed25519, derivación de
   OwnerId).
3. **Validación de entrada no confiable** (decodificación wire,
   límites, panics).
4. **DoS de recursos** (caché DNS, mempool, DHT, sync).
5. **Determinismo/consenso** (reorg, GC, resurrección de estado).

## 1. Auditoría estática automatizada (día 1 — mecánica)

- `cargo audit` (CVEs de dependencias: libp2p/quinn/rustls/redb/ed25519).
- `cargo geiger` + revisión de `unsafe` (esperado ≈ 0).
- `cargo clippy --workspace --all-targets -- -D warnings`
  (ya en verde, mantener como barrera de CI).
- Fuzzing de las fronteras de decodificación (días 2-3):
  `cargo-fuzz` sobre `scone_protocol::decode_complete` (Message,
  Transaction, Block, SignedDnsRecord, Checkpoint) — esta es LA
  superficie donde «nunca hacer panic con datos no confiables» debe
  cumplirse. Presupuesto: 24 h de fuzzing continuo, corpus sembrado
  desde los tests existentes. Criterio de salida: cero crashes, cero
  timeouts, cobertura de las ramas de decodificación > 95%.
- Fuzzing del parser wire DNS (dns.rs `parse_query`,
  `extract_answers`) y del parser de archivos de registros
  (domain.rs).

## 2. Revisión manual dirigida por componente (días 4-8)

Ordenada por criticidad del activo, 1 componente = 1 checklist:

### 2.1 scone-crypto + derivación de identidad (día 4)
- OwnerId = BLAKE3(pubkey): comprobar la canonización de bytes
  (ninguna representación ambigua → 2 claves, 1 propietario).
- Ed25519: DESERIALIZACIÓN estricta (¿se rechaza la clave no
  canónica?), verificación con separación de contextos (cada uso
  tiene su etiqueta de dominio: SCONE-TX-SIG-V1, SCONE-BLOCK-V1,
  registro DHT — imposible el replay entre protocolos).
- Keystore: passphrase → clave (¿KDF? ¿iteraciones?), zeroización,
  permisos de archivo, oráculo de padding sobre el cifrado.

### 2.2 Aplicación de transacciones (día 5) — el núcleo
Para CADA variante de Transaction (13 hoy):
- ¿Es la tx idempotente bajo replay (txid único, nonce/secuencia)?
- ¿La comprobación de propiedad precede a cualquier efecto?
  (signer == owner aprendido on-chain, nunca confiado desde la
  propia tx.)
- Secuencia de UpdateDomain: exactamente current+1 (no ≥) —
  comprobar el rechazo de huecos y el rechazo de replays.
- Renew: cotas temporales (≤ 3 años, > now) — el tiempo viene del
  BLOQUE, nunca del reloj local (regla ya fijada, comprobar cada
  punto de llamada).
- AssignDomain/TransferTld: ¿la pubkey del asignado se APRENDE
  (la trampa del pool sin pubkey — cf. skill) en todas las rutas.
- GC/expiración: imposible la resurrección (la paradoja
  precheck-OK/lookup-OK/push-falla) — comprobar
  gc_expired_journaled.

### 2.3 Capa de red (día 6)
- Cada byte remoto pasa por un decodificador estricto y acotado
  ANTES de asignar (MAX_* de limits.rs) — buscar `Vec::with_capacity(input_len)`
  sin cota.
- requery/sync GetBlocks: cotas por par y por ciclo (ya:
  128 bloques/petición) — comprobar la ausencia de bucles de
  re-descarga.
- Gossip Record: reverificación contra la cadena ANTES de
  caché/gossip (verificado por mis tests record_dht) — comprobar
  también el tope de fanout.
- kad DHT: ¿puede un par hostil forzar el almacenamiento de N
  registros (¿cuota por par?) o el desalojo selectivo de registros
  honestos?

### 2.4 Superficie DNS (día 7)
- Caché: inyección imposible (clave canónica (qname,qtype) — sin
  colisión de mayúsculas/codificación), presupuesto duro de memoria
  (probado), sin envenenamiento de caché vía la ruta de fallback
  (TXID + eco de la pregunta — verificar que el upstream no puede
  falsificarse en remoto).
- Respuestas: AA honesto, TC honesto, sin amplificación (ratio
  respuesta/petición acotado — MAX_INFLIGHT_UDP ya está, comprobar
  la réplica TCP).

### 2.5 Almacenamiento y génesis (día 8)
- redb: corrupción de disco → error tipado, sin panic (el test
  diskfail existe en el lado del firmante — ¿el equivalente en el
  relay?).
- Génesis: el hash cubre todo el manifiesto; dos génesis distintos
  = rechazo del data-dir (probado); verificar la imposibilidad de
  inyectar un génesis A POSTERIORI sobre un data-dir sincronizado.

## 3. Tests adversariales (días 9-12) — escribir lo que falte

Para cada hallazgo de la revisión manual, un test que lo demuestre:
- nodo hostil en la sim (SimNet con un nodo que envía mensajes
  malformados/disconformes) — extender SimNet con un rol «bizantino
  pasivo»;
- malla real de 30 nodos con 3 nodos hostiles (bytes aleatorios,
  ráfagas de txs inválidas, registros DHT falsificados) — la
  batería de caos existe, añadir el rol de atacante;
- verificaciones cruzadas: dos implementaciones de la decodificación
  (el doc protocol.md es NORMATIVO — un auditor implementa el wire
  a partir del doc SOLO y compara bytes; cualquier divergencia =
  bug del doc O del código).

## 4. Auditoría externa (si hay presupuesto) — tras 1-3

Alcance mínimo rentable para un proveedor: scone-crypto,
scone-protocol (wire), aplicación de txs (state.rs), ~8-10 k LOC.
Formato: 2 semanas, 1 auditor cripto senior + 1 revisor. Entregable:
informe con severidad, PoCs, correcciones revisadas. Apoyarse en la
documentación normativa existente (docs/technical) como base de la
especificación de requisitos — una ventaja de coste real.

## 5. Criterios de salida (go/no-go de la mainnet)

- [ ] cargo-fuzz 24 h sin crash sobre los 5 decodificadores
- [ ] Revisión 2.2 completa: cada variante de tx tiene su ficha
      (propiedad/replay/tiempo/efectos) validada por el revisor
- [ ] Malla de 30 nodos incluyendo hostiles: 48 h sin divergencia
      de tip ni corrupción de caché
- [ ] Divergencia doc/código: cero (verificada por la doble
      implementación wire)
- [ ] CVEs bloqueantes: cero (cargo audit)
- [ ] Auditoría externa del ámbito cripto/wire/txs hecha (si está
      presupuestada)

## Orden de ejecución recomendado

1→2.1→2.2 (el núcleo, 60% del riesgo)→2.3→2.4→2.5→3 en paralelo
con cada revisión (el test adversarial se escribe el día del
hallazgo, no después).
