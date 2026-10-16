# RFC — Prueba de operación y comité de finalidad (proof-of-service)

> Estado: NORMATIVO — implementado de extremo a extremo (pasos 1 a
> 5: core/protocol/blockchain/network/CLI; este documento es la
> forma consolidada). Cada regla de abajo está verificada contra el
> código; las constantes citadas son los valores codificados.
> Fecha: octubre de 2026 (consolidación normativa). Toca:
> `scone-core`, `scone-protocol`, `scone-blockchain`,
> `scone-network`, `scone` (CLI).

## 1. Motivación

El pool de finalidad es el pool de propietarios (propietarios de
dominios/TLDs vivos). Poseer otorga un asiento — **operar** no
otorga nada. Este RFC liga el derecho a firmar checkpoints a la
prueba de operación de un relay en línea, sin condicionar jamás la
propiedad:

- **Libertad preservada**: un TLD privativo sigue siendo asignable
  sin condición de actividad mientras su plazo se renueve
  (`RenewTld`, plazo de 3 años). Solo la *firma de checkpoints* se
  gana con la presencia.
- **Incentivo a operar**: operar un TLD privativo (o cualquier
  asiento) = sentarse en el comité. El operador participa en
  mantener la red en pie y el protocolo lo reconoce.
- **Descentralización medible**: el comité se convierte en el
  conjunto de operadores *realmente en línea*, no de claves
  dormidas.

## 2. Visión general

Una transacción `Heartbeat` ligera (discriminante `0x71`), firmada
por una clave de **operador**, emitida una vez por época. El estado
mantiene una ventana deslizante de heartbeats recientes. El comité
de una época se elige **desde el subconjunto del pool cuyo
operador efectivo tiene un heartbeat fresco**. Sin heartbeat: sin
firma, pero sin pérdida de derechos de propiedad.

## 3. La transacción Heartbeat

```
Heartbeat {
    operator: PublicKey,     // clave Ed25519 del operador (el firmante)
    owner: OwnerId,          // identidad RE-DERIVADA del operator (nunca confiada)
    tip_seen: [u8; 32],      // último tip visto por el nodo
    height_seen: u64,        // altura correspondiente
    timestamp: u64,          // segundos Unix
} + firma Ed25519 sobre el payload canónico (red inyectada)
```

Cuerpo firmado (wire, `scone-protocol`):
`operator[32] ‖ owner[32] ‖ tip_seen[32] ‖ height_seen.v ‖
timestamp.v ‖ operator[32]` (la clave del firmante aparece en la
cola del cuerpo, como en toda tx) `‖ signature[64]`, es decir
~230 bytes codificados — **sin PoW**: el coste es la presencia
continua (un heartbeat por época, durante meses).

El firmante ES el operador: `owner` siempre se re-deriva de
`operator` (nunca tomado de la entrada), y la decodificación
rechaza con un error tipado (`InvalidOwner`) una clave de firma
distinta del campo `operator` de la cabeza.

- `tip_seen`/`height_seen` sirven a la observabilidad (detección de
  operadores con bugs o rezagados); NO influyen en la validez (un
  nodo ligeramente rezagado sigue siendo un nodo vivo).

### 3.1 Validación pura (scone-core)

1. Firma Ed25519 válida sobre el payload canónico de la red.
2. `owner` == identidad derivada de `operator` (`InvalidOwner` en
   caso contrario) — comprobado en codificación, en decodificación
   Y por `validate()`.

### 3.2 Reglas de estado (aplicación, scone-blockchain)

Al aplicar un heartbeat en un bloque portador de altura
`block_height` (todo rechazo precede a cualquier mutación):

1. **Anti-replay entre épocas**: `timestamp` estrictamente
   creciente por operador — rechazo tipado `StaleHeartbeat` si
   `timestamp <= operator_last_hb[operator]`.
2. **Unicidad por época**: solo un heartbeat aceptado por
   (`operator`, época del bloque portador) — el duplicado se
   rechaza con un error tipado (`HeartbeatDuplicate`). La época es
   `block_height / epoch_min_blocks` (misma definición que la
   finalidad, nunca una constante nueva).
3. Inserción journaled en la ventana + actualización de
   `operator_last_hb` (reversible en reorg, undo bit-exacto).

### 3.3 Anti-spam

- 1 tx aceptada por operador por época — el flujo máximo es
  `pool_size / epoch_secs`, despreciable.
- Sin tasas (pre-economía): el ritmo está estructuralmente
  acotado.

## 4. Delegación de operación

El enlace asiento → operador pasa por dos transacciones dedicadas:

- `SetTldOperator { tld_id, operator }` (discriminante `0xA9`),
  firmado por el propietario **actual** del TLD **privativo** (un
  TLD abierto no tiene nadie que firme legítimamente — rechazo
  tipado `TldIsOpen`). `operator: None` = revocación, vuelta a la
  auto-operación por la clave del propietario; revocable al
  instante.
- `SetDomainOperator { domain_id, operator }` (discriminante
  `0x5E`), firmado por el propietario actual del dominio. Misma
  semántica.
- `RegisterTld` / `RegisterDomain` / `AssignDomain` NO llevan campo
  `operator`: el enlace asiento → operador es siempre un hecho
  on-chain reciente, resuelto en el tip.

La opción de operador en el wire es canónica: `0x00` = `None`
(auto-operación), `0x01` = clave cruda de 32 bytes; cualquier otro
valor se rechaza en decodificación.

El estado mantiene:

```
seat_operators: HashMap<OwnerId, OperatorKey>   // por asiento
operator_seats: HashMap<OperatorKey, u64>       // contador acotado
operator_last_hb: HashMap<OperatorKey, u64>     // último ts aceptado
```

Reglas de aplicación (`apply_seat_operator`, journaled):

- **Idempotencia**: re-delegar al mismo operador (o revocar una
  delegación inexistente) es un no-op sin journal (el undo
  bit-exacto lo exige).
- **Cota anti-alquiler**: un operador solo puede ser delegado por
  **un único asiento** a la vez (`MAX_SEATS_PER_OPERATOR = 1`). Un
  segundo asiento que delegue a un operador ya ocupado → rechazo
  tipado `OperatorAlreadyDelegated`. Justificación: sin cota, un
  gran operador vende la firma al por mayor y recrea la
  centralización — exactamente lo que el comité electo debe evitar.

## 5. Ventana de frescura

### 5.1 Ventana deslizante

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- Entrada insertada al aplicar un heartbeat válido (§3.2).
- Purga determinista en CADA aplicación de bloque
  (`gc_heartbeats_journaled`): toda entrada de época `e` con
  `e + HEARTBEAT_WINDOW_EPOCHS <= epoch` se elimina (journaled —
  reversible en reorg, como las colas de expiración).
- `HEARTBEAT_WINDOW_EPOCHS = 2`: la época actual + la anterior —
  un operador puede perder UNA época sin perder frescura, la
  segunda lo caduca.
- **Tope estructural de memoria**: la ventana contiene como mucho
  `HEARTBEAT_WINDOW_EPOCHS × operator_count` entradas (una por
  operador por época, clave tupla); `operator_last_hb` está
  acotado por el número de operadores y nunca se purga (la regla
  anti-replay es monótona).

### 5.2 Frescura

«Fresco» = presencia de una entrada `(e, operator)` con `e` en
`[epoch − HEARTBEAT_WINDOW_EPOCHS + 1, epoch]`. Como el estado
purga en cada bloque, la presencia en `recent_heartbeats` ES la
frescura (`operator_is_fresh`).

### 5.3 Operador efectivo

El **operador efectivo** de un asiento es `seat_operators[seat]` si
existe una delegación viva, si no la clave del propietario
(`effective_operator`). El asiento firma bajo esa clave efectiva —
una delegación mueve el derecho de firma, nunca la propiedad.

## 6. Elección del comité

```
eligible = { operadores efectivos de los asientos del pool congelado
             (clave expulsada excluida) con heartbeat fresco }

if |eligible| >= MIN_FINALITY_COMMITTEE_SIZE (= 4) :
    committee = select_committee(seed, eligible)   // sorteo existente
else :
    committee = select_committee(seed, pool)       // degradación segura
```

Detalles normativos (verificados en `finality.rs`, especificados en
`/docs/en/technical/blockchain.md` § «Asientos y el comité de
finalidad»):

- el `pool` es el pool **congelado en el último checkpoint
  finalizado** (`FinalizedBase.pool` — nunca el estado vivo): una
  vista local divergiría entre nodos y el quórum nunca se
  formaría;
- la `seed` deriva del hash de contenido del checkpoint anterior
  (`signing_hash`);
- cada asiento del pool congelado se resuelve a su operador EFECTIVO
  por el estado en el tip (delegación y heartbeats son hechos
  on-chain recientes: dos nodos en el mismo tip computan el mismo
  conjunto); las claves expulsadas siguen excluidas;
- el sorteo corre sobre el conjunto de claves efectivas
  ordenado/deduplicado (`select_committee` sin cambios).

**Degradación explícita y segura**: una red en bootstrap (sin
heartbeat), o una caída general de operadores, cae al pool entero
— la finalidad nunca se pierde por construcción. Es la regla
«nunca interbloquear» del proyecto aplicada aquí.

### 6.1 Bootstrap en frío

Época 0 (y mientras menos de `MIN_FINALITY_COMMITTEE_SIZE`
operadores estén frescos): el pool entero. La restricción se
endurece naturalmente a medida que los operadores emiten. Sin
período transitorio especial que configurar.

## 7. Emisión automática (scone-network)

En cada ciclo de producción (`Wake::Produce` — sin temporizador
nuevo), el relay comprueba si se cruzó un LÍMITE DE ÉPOCA
(`height / epoch_min_blocks`, misma definición que la finalidad).
En el primer ciclo de una época nueva, emite un heartbeat si:

1. una clave anchor está armada (`--anchor-key` o auto), Y
2. esa clave misma ostenta un asiento vivo (comprobación LOCAL: la
   clave está en el pool de elegibilidad on-chain — propietarios de
   dominios/TLDs vivos, claves expulsadas excluidas).

Anti-spam local: como mucho UNA emisión por época por nodo (memo
`last_emitted_epoch`, memoria local, nunca estado de cadena); el
memo solo se toma tras una inyección exitosa en el mempool — un
fallo reintenta en el ciclo siguiente. Nada aquí es fatal: en el
peor caso el heartbeat espera al ciclo siguiente.

Un **operador delegado** (que no es él mismo propietario) no está
cubierto por la emisión automática: emite vía el constructor CLI de
la §8 (operación manual o en rutina aislada). El heartbeat NO es
necesariamente producido en un bloque por el emisor mismo: es una
tx como cualquier otra (mempool, producción por cualquiera).

## 8. CLI (`scone`)

El constructor offline de heartbeat (retirado con la superficie
`tx`) fue eliminado con la superficie tx offline
(cli-simplify-v1). Los relays emiten su propio heartbeat
automáticamente (§7); la emisión manual/enrutina aislada, si algún
día volviera a hacer falta, volvería como comando dedicado. Los
valores por defecto eran: `tip_seen` = 32 bytes cero (solo
observabilidad), `height_seen` = 0, `timestamp` = now.
- Como en toda tx, `tx sign` **re-liga** operator y owner a la
  clave de firma REAL (los valores de entrada nunca se confían);
  `tx verify` valida localmente.

## 9. Observabilidad

- `NodeStatus` (mensaje p2p wire `0x0d`) lleva
  `fresh_operators: u32` (varint): el número de operadores con un
  heartbeat fresco — el subconjunto vivo del pool elegible para el
  comité (operadores efectivos de los asientos del pool, claves
  expulsadas excluidas, ordenado/deduplicado). Lo rellena el relay
  desde `state().fresh_operators(height)`; lo muestra `scone
  status` como `fresh operators: N`.
- Argumento de campaña: «operar un TLD privativo = sentarse en el
  comité de finalidad del DNS descentralizado» — verificable
  on-chain.
- Superficie futura (sin especificar): `operator_status` — por
  asiento, el operador delegado, la frescura (fresco / caducado /
  nunca visto), el último `tip_seen`. El RPC local de control fue
  eliminado (nodo solo-p2p); la superficie queda por definir.

## 10. Lo que este RFC NO hace

- **Sin pérdida de posesión** por inactividad (la opción A —
  decaimiento del TLD no operado — está explícitamente rechazada).
- **Sin recompensa** (la opción C espera una capa económica).
- El heartbeat no prueba la *calidad* del servicio (bloques
  producidos, registros DHT servidos) — solo la presencia. Contar
  la contribución real es una posible extensión futura, más
  manipulable.

## 11. Invariantes protegidos

- Determinismo bit-exacto: la ventana y la elección son funciones
  puras del estado aplicado (journaled, undo bit-exacto).
- Nunca panic con datos de red: heartbeat o delegación malformados
  → rechazos tipados (`InvalidOwner`, `StaleHeartbeat`,
  `HeartbeatDuplicate`, `OperatorAlreadyDelegated`, `TldIsOpen`,
  `NotTldOwner`, `NotOwner`).
- La posesión es incondicional; solo la firma de checkpoints es
  condicional.

## 12. Constantes y parámetros (valores codificados)

| Nombre | Valor | Ubicación |
|---|---|---|
| `HEARTBEAT_WINDOW_EPOCHS` | 2 | `scone-blockchain/src/state.rs` |
| `MAX_SEATS_PER_OPERATOR` | 1 | `scone-blockchain/src/state.rs` |
| `MIN_FINALITY_COMMITTEE_SIZE` | 4 | `scone-core/src/consensus_params.rs` |
| Discriminante `Heartbeat` | `0x71` | `scone-protocol/src/transaction.rs` |
| Discriminante `SetTldOperator` | `0xA9` | ídem |
| Discriminante `SetDomainOperator` | `0x5E` | ídem |

Estos valores forman parte del consenso: cambiarlos rompe la
re-jugabilidad de la cadena (ventana/purga) o la elección
(comité).

## 13. Plan de implementación — estado final

| Paso | Crate | Contenido | Estado |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`, `SetTldOperator`, `SetDomainOperator`, wire, validación | hecho |
| 2 | blockchain | ventana deslizante (journaled), pool filtrado, elección, ~15 tests | hecho |
| 3 | network | emisión automática, relay de tx | hecho |
| 4 | CLI | `tx build heartbeat`, muestra `fresh operators` en `scone status` | hecho |
| 5 | docs | este RFC consolidado en forma normativa, espejo fr/en | hecho |
