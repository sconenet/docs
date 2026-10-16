# Modelo de seguridad — selección de comité y anti-grinding

> Documento normativo. Cualquier divergencia doc/código = bug.

## Visión general

El comité de la época N+1 se sortea a partir de la semilla
`next_seed(signing_hash(checkpoint N)) = blake3("SCONE-SEED-V1" || signing_hash(checkpoint N))`,
sobre el **pool de elegibilidad congelado en el bloque finalizado
por el checkpoint N**. `signing_hash` es el hash del **contenido**
del checkpoint (época, altura, bloque, prev, raíz de estado — juego
de firmas excluido): la semilla es una función pura de QUÉ se
finalizó, nunca de QUÉ subconjunto de quórum lo firmó — dos vistas
que contienen subconjuntos distintos (ambos válidos) del mismo
checkpoint eligen el mismo comité.

## Amenaza: hacer grinding de la semilla del comité

Un atacante quiere influir en la composición del comité. Existen
tres ventanas de ataque; las tres están cerradas estructuralmente:

### 1. Antes de la semilla — imposible moldear la propia posición

La semilla de la época N+1 deriva del **hash del checkpoint N**,
firmado por quórum (floor(2n/3)+1 del comité N — `quorum_for`). El
checkpoint N compromete el `state_root` de un estado que **fija el
pool** (base PoS = estado exacto en el bloque finalizado,
`FinalizedBase.pool`). Producir una semilla distinta requeriría un
checkpoint distinto firmado por quórum — es decir, corromper ≥ 1/3
del comité N, que es la hipótesis de fallo excluida por el modelo
BFT.

### 2. Después de la semilla — demasiado tarde para unirse

Cualquier registro (reclamación de dominio/TLD, aparición de una
nueva clave de propietario) solo cuenta para épocas posteriores,
cuyas semillas aún no se conocen. El pool queda congelado en
`FinalizedBase` ANTES de que nadie (incluidos los firmantes del
checkpoint N) pueda computar la semilla de N+1.

### 3. Selección «best-of-m» sin ventaja

La posición de una clave en el sorteo =
`blake3("SCONE-COMMITTEE-V1" || seed || pk)` (uniforme), cuyo único
grado de libertad es `pk`. Quien haga grinding generando m claves y
registre solo la mejor posicionada debe primero **conocer la
semilla** — pero en ese momento el pool ya está congelado.
Registrar m candidatos «por si acaso» cuesta m pruebas de trabajo
(una por dominio/TLD) y produce exactamente m/|pool| presencia
esperada: **ninguna ventaja sobre m registros directos**. La
estrategia adaptativa (ver la semilla, luego elegir qué registrar)
es temporalmente imposible.

### 4. Sesgo del último productor (ataque de retención)

El productor del último bloque antes de un checkpoint podría
retener su bloque para influir en `signing_hash(checkpoint N)` y por
tanto en la semilla. Dos salvaguardas:

- el checkpoint firma un estado a la altura H, y el **pool se
  evalúa en el timestamp del bloque H** — retener un bloque retrasa
  el checkpoint pero no cambia el pool de una época ya congelada
  por el checkpoint N−1 (los pools se solapan poco de época a
  época);
- retener le cuesta al atacante su propia recompensa de producción
  y solo afecta a la LIVENESS (checkpoint retrasado), nunca a la
  SAFETY: dos checkpoints finales incompatibles requerirían ≥ 1/3
  dobles firmas, demostrables con `SlashTx` y castigadas con
  expulsión.

Conclusión: un VRF o un commit-reveal no aportarían nada aquí — la
semilla ya es función de un evento firmado por quórum que es
impredecible-retrasable-pero-infalsificable, y la congelación del
pool elimina la única palanca adaptativa. No complicar el protocolo
(restricción P1.7: «si es necesario» — análisis: no es necesario).

## Equivocación y slashing

Un anchor que firme dos checkpoints conflictivos en la misma época
puede ser demostrado por cualquiera que posea ambas firmas
(`SlashTx`, discriminante `0x15`): expulsión del pool PoS
(journaled, reversible en reorg). Los dominios/TLDs del acusado no
se confiscan (sanción mínima — decisión M9).

## Propiedades verificadas por tests

- `checkpoint_hash_engages_signatures`: el hash del checkpoint
  cubre las firmas (sin manipulación a posteriori);
- `seed_chain_and_recovery`: la cadena de semillas deriva
  exactamente, recuperación k incluida;
- `small_committee_never_finalizes`: guarda BFT — un comité por
  debajo de `MIN_FINALITY_COMMITTEE_SIZE` nunca finaliza;
- e2e 4 relays: un checkpoint finaliza y se propaga, misma época
  en todos los nodos.

## Límites conocidos (documentados, sin corregir)

- El pool elegible incluye a los propietarios de TLDs vivos
  (reclamación = PoW gastado) — superficie Sybil acotada por el
  coste PoW por clave;
- El `state_root` del checkpoint se verifica en RAM solo cuando el
  checkpoint apunta al tip; el replay retroactivo es una ruta de
  almacenamiento/relay (ver `docs/technical/blockchain.md`).
