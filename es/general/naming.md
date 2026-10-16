# Nomenclatura

## Alfabeto

Letras ASCII minúsculas, dígitos y guiones (LDH, RFC 1123):
`[a-z0-9-]`, con el guion ni en primera ni en última posición de una
etiqueta. Sin mayúsculas, sin guion bajo.

La validación es **estricta** para ASCII: un nombre no canónico se
rechaza, nunca se corrige. `"ABC"` → error (sin paso a minúsculas
silencioso). Justificación: una frontera de protocolo debe ser
determinista; la canonización implícita crea ambigüedades de
identidad.

## Nombres internacionalizados (IDNA)

Los nombres no latinos se aceptan como **entrada** y se codifican en
punycode (IDNA, RFC 5890/5891, UTS#46): `例子.uip` tiene
`xn--fsqu00a.uip` como forma canónica.

- la forma canónica (on-chain, ids BLAKE3, DNS, almacenamiento) es
  siempre la **A-label** ASCII: todas las reglas LDH anteriores se
  aplican a esa forma;
- la forma unicode original (**U-label**) se conserva en memoria para
  la visualización (`DomainName::unicode()` / `TldName::unicode()`);
- el límite de 63 bytes por etiqueta y el límite total de 253 bytes
  se aplican a la forma canónica A-label, no al UTF-8 crudo;
- ASCII se queda en la ruta estricta: IDNA solo se aplica a la
  entrada no ASCII (`"ABC"` sigue siendo inválido);
- política anti-homoglifos (scripts mixtos, confundibles): sin
  filtrar por ahora, solo UTS#46.

## TLD

Un TLD es una etiqueta DNS como cualquier otra: el límite genérico
de etiqueta (RFC 1035: 63 bytes) se aplica, no un límite más corto
específico de TLD.

- patrón: `[a-z0-9-]{1,63}` (LDH, sin guion inicial/final), sobre la
  forma canónica A-label
- límite codificado: `TldName::MAX_LEN = 63` bytes
- válidos: `uip`, `com`, `test`, `abc12`, `0x`, `a-b`, un TLD de
  exactamente 63 bytes, `éxemple` (→ `xn--exemple-2ua`), `例子`
  (→ `xn--fsqu00a`)
- inválidos: un TLD cuya A-label supere 63 bytes, `ABC`, `-ab`,
  `ab-`, cadena vacía

## Dominios

- forma: `label[.label]*.tld` — al menos dos etiquetas (`name.tld`);
- cada etiqueta: LDH `[a-z0-9-]{1,63}`, guion ni inicial ni final;
- longitud total: ≤ 253 bytes;
- sin etiquetas vacías (`a..b`, `.a`, `a.` inválidos);
- la última etiqueta debe ser un TLD válido.

### Subdominios

El tipo `DomainName` acepta la jerarquía completa desde el
principio:

```text
example.uip
shop.example.uip
api.shop.example.uip
```

Un subdominio tiene un `DomainId` **distinto** del de su padre.
Las reglas de asignación de subdominios (derechos del propietario
del padre, etc.) son trabajo futuro.

## Canonización

Un nombre validado ya es canónico: `canonical() == as_str()`.
Por tanto, se rechaza hoy: mayúsculas, puntuación distinta del
separador `.`, espacios, unicode, caracteres invisibles, etc.

## DomainId

```text
DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || canonical_name)
```

- determinista: mismo nombre canónico ⇒ mismo id, en todas partes,
  para siempre;
- inyectivo en la práctica: dos nombres distintos ⇒ dos ids
  distintos;
- el nombre crudo **nunca** debe servir como identificador interno
  (claves de la blockchain, claves DHT) — solo para entrada y
  visualización;
- el prefijo `SCONE-DOMAIN-V1` aísla los dominios de derivación:
  cambiar la regla (alfabeto, normalización) equivale a crear un
  nuevo espacio de nombres versionado (un nuevo prefijo de
  derivación), sin reescribir el existente.

### Representación textual

Codificación hexadecimal en **minúsculas** (`0-9a-f`) de los 32
bytes crudos: exactamente 64 caracteres, sin prefijo (`0x`), nunca
truncado.

```text
DomainId([0xab; 32]) → "ababababababababababababababababababababababababababababababababab"
DomainId([0x00; 32]) → "0000000000000000000000000000000000000000000000000000000000000000"
```

Esta forma la produce `Display` (el `Debug` de Rust imprime
`DomainId(<hex>)`, misma cadena). Cualquier salida consumida por un
humano o una máquina fuera del formato binario canónico — logs,
mensajes de error, salida del CLI — DEBE usar esta representación y
ninguna otra (sin truncamiento, sin prefijo, sin mayúsculas).

## Límites y reglas futuras (no decididas)

- nombres internacionalizados (IDN) vía punycode;
- longitud mínima de las etiquetas de segundo nivel;
- expiración/renovación de las reclamaciones de dominios.

## TLDId (M7a)

Un TLD registrable lleva su propia identidad, distinta de la de
cualquier dominio:

```text
TldId = BLAKE3-256("SCONE-TLD-V1" || tld)
```

- el prefijo `SCONE-TLD-V1` es **distinto** de `SCONE-DOMAIN-V1`:
  los espacios de ids de TLD y de dominio son disjuntos por
  construcción — registrar el TLD `uip` (una reclamación sobre el
  espacio de nombres `*.uip`) nunca puede usurpar la identidad de un
  dominio, ni al revés;
- mismas reglas de visualización que `DomainId`: hex minúsculas, 64
  caracteres, nunca truncado;
- vector fijado (derivación fijada por test):
  `TldId("uip") = ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e`.

La transacción `RegisterTld` (reclamación firmada de un TLD,
`owner` siempre recompute a partir de la clave embebida) forma parte
del enum `Transaction` y del formato wire desde M7b: discriminante
`0x93` (constante opaca arbitraria), payload firmado
`SCONE-TX-SIG-V1`, reglas completas en
`/docs/en/technical/transactions.md`.
El `RegisterDomain` de un dominio, por su parte, lleva el nombre
canónico en claro.

## Reserva/gobernanza de TLDs

Abierto por M7a: `RegisterTld` reclama un TLD libre en beneficio de
la identidad derivada de la clave firmante. Desde M8b la gobernanza
está completa: reclamación con PoW (dificultad por red) que abre un
plazo renovable de 3 años (`RenewTld`), espacio de nombres
**cerrado** por defecto (solo asignación), apertura explícita
(`SetTldOpen`), transferencia (`TransferTld`), abandono
(`RevokeTld`, reclamable de nuevo). Los TLDs privativos viven 3 años
(gracia de 60 días, gestión congelada durante la gracia); los TLDs
abiertos llevan un piso de 1 año mantenido por sus dominios; los
dominios expiran al año (renovable, gracia de 30 días — ver
`/docs/en/technical/blockchain.md`).

**TLDs raíz de la ICANN**: reservados. `RegisterTld` sobre un TLD de
la lista IANA fijada (`is_icann_tld`, ~1.440 entradas + RFC 2606)
se rechaza con un error tipado (`IcannTldReserved`) — la raíz
legada pertenece al DNS existente. En resolución, estos nombres se
reenvían a los upstreams configurados (`--dns-upstream`) o se
rechazan con REFUSED sin upstream. Los TLDs añadidos por ICANN
posteriormente siguen siendo reclamables (lista fijada: coherencia
> frescura).


## Economía del ciclo de vida — quién paga qué y cuándo

El modelo de autorización sigue la naturaleza del recurso: el
trabajo (PoW) paga lo que entra en el común, una firma (PoS)
gobierna lo que pertenece a alguien. Resumen normativo:

### TLD privativo

- **Nacimiento — PoW**: reclamado por prueba de trabajo, dificultad
  por longitud (mainnet: 1 carácter ≈ 34 d de computación; piso ≈
  4,3 d a 10+ caracteres). El reclamante se convierte en
  propietario y ostenta un **asiento PoS** (pool de elegibilidad de
  validadores — heartbeats obligatorios para mantenerse fresco,
  delegación de operador disponible).
- **Vida — PoS**: la firma del propietario autoriza todo en el
  espacio de nombres: `assign` (conceder un dominio), `set-tld-open`
  (irreversible — **libera el asiento**), `transfer`, `revoke`, y
  la renovación de los dominios que viven bajo él.
- **Plazo — 3 años**: la reclamación abre un plazo FIJO de 3 años
  (`TLD_TERM_SECS`). **Renovación — PoW igual a la creación**
  (`RenewTld`, misma curva de dificultad por longitud; reto ligado
  a la expiración actual — anti-reutilización): cada plazo de vida
  se paga en trabajo.
- **Expiración — gracia de 60 días, gestión congelada**: pasada la
  fecha límite, una gracia de 60 días (`TLD_GRACE_SECS`) mantiene
  posible la renovación, pero toda operación de gestión distinta de
  la renovación queda congelada — notablemente `assign`: un
  propietario caducado pierde los derechos de gestión hasta que
  renueve. Los dominios ya asignados debajo viven su propia vida
  (su propio `valid_until`) — el TLD se congela, no sus hijos.
- **Muerte — GC**: plazo + gracia transcurridos sin renovación → el
  TLD desaparece (asiento PoS liberado, reclamable por cualquiera
  con un PoW fresco). Un `TransferTld` transfiere el plazo restante
  (sin reinicio).

### TLD abierto (nacido abierto)

- **Nacimiento — PoW reducido** (un tercio más barato que el
  privativo: un recurso común que nadie gobierna). **Sin asiento
  PoS**.
- **Piso — 1 año** (`OPEN_TLD_FLOOR_SECS`): al nacer (o al abrirse
  vía `SetTldOpen`), un piso de 1 año impide el crear-destruir
  instantáneo de un TLD sin dominios.
- **Vida**: nadie firma nada — los dominios se registran en
  autoservicio. Cada dominio vivo debajo retrasa el reloj de muerte
  (sigue la expiración del dominio más reciente, acotada por el
  piso).
- **Muerte — GC** cuando ya no vive ningún dominio debajo y el
  piso/expiración ha pasado; el nombre vuelve a ser reclamable.

### Dominio bajo un TLD abierto

- **Nacimiento — PoW** (curva reducida frente al TLD: mainnet
  1 carácter ≈ 8,5 d, piso ≈ medio día).
- **Renovación — PoW idéntica al nacimiento** (misma curva de
  dificultad; el reto está ligado a la expiración actual, y por
  tanto es distinto del registro y de cada renovación anterior):
  cada plazo de vida se paga en trabajo, un dominio no puede
  «recogerse» gratis (con ≤ 3 años de antelación).
- **Muerte**: expiración → gracia de 30 d (renovable) → GC → el
  nombre se vuelve reclamable por **cualquiera** con un PoW fresco.

### Dominio bajo un TLD privativo

- **Nacimiento — el PoS del propietario del TLD**: solo el
  propietario (o su operador delegado) concede (`assign`) — la firma
  es la autoridad, sin PoW.
- **Renovación — el PoS del propietario del TLD**: firmada por el
  propietario del TLD, campo `proof` vacío.
- **Muerte**: gracia y luego GC; la nueva reclamación pasa de nuevo
  por el `assign` del propietario.

### Invariantes transversales

- Cada PoW lleva salt de **época**, con una ventana de validez
  **hacia atrás** proporcional a la dificultad (un minado de varias
  semanas aún aterriza aunque la época haya cambiado).
- La equivocación del firmante PoS se **pena** (slashing; prohibición
  vitalicia — M9).
- Cada operación reverifica todo en el momento de aplicar: nada se
  da por sentado (hashes, pruebas y firmas recomputados).
