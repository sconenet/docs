# Keystore — especificación del formato del archivo de claves `.sconekey`

> Documento normativo. Cualquier divergencia entre este documento y
> `crates/scone-keystore/src/lib.rs` es un bug (identificar la
> fuente de verdad y corregirla). Referencia de implementación:
> crate `scone-keystore` (v2).

## Propósito

Guardar en disco una clave privada Ed25519 (la *semilla*, 32 bytes),
cifrada con una passphrase. La clave **nunca** toca disco en claro;
todos los secretos (passphrase, material derivado) se ponen a cero
en memoria tras su uso.

## Visión general

- **KDF**: Argon2id (v0x13), parámetros por defecto `m_cost` =
  64 MiB, `t_cost` = 3, `p_cost` = 1 (valores por defecto del crate
  `argon2`).
- **Cifrado**: XChaCha20-Poly1305 (AEAD), clave de 32 bytes derivada
  del KDF, nonce aleatorio de 24 bytes por archivo.
- **Contenido cifrado**: exactamente la semilla Ed25519 (32 bytes);
  el tag Poly1305 (16 bytes) se guarda como sufijo del texto
  cifrado (salida estándar de la API `encrypt` del AEAD).

## Sellado en bytes (módulo `bytes`, web)

`scone_keystore::bytes::{seal_to_bytes, open_from_bytes}` produce y
consume el MISMO formato sin tocar el sistema de archivos: el
cliente web genera la identidad en el navegador (`scone-crypto` +
bip39) y sella los bytes exactos del archivo de claves
(descargable, abrible por el CLI tal cual — test de
interoperabilidad en la suite). La ruta nativa
(`create`/`create_with_phrase`) delega en el mismo sellado: una
sola codificación del formato, tres destinos.

## Formato binario (v2) — cabecera de 52 bytes + metadatos públicos + texto cifrado

```text
Offset  Tamaño      Campo
0       9           magic: "SCONEKEY1" (ASCII — sin cambios desde v1)
9       1           version: u8 = 2
10      1           m_cost_mib: u8 (m_cost en MiB; m_cost_kib = valor × 1024)
11      1           t_cost: u8 (iteraciones Argon2id)
12      16          salt Argon2id (aleatoria por archivo)
28      24          nonce XChaCha20-Poly1305 (aleatorio por archivo)
52      32          public_key: clave pública Ed25519 (32 bytes, EN CLARO)
84      8           created_at: u64 big-endian (Unix, segundos)
92      1           label_len: u8 (0..=255)
93      label_len   label: UTF-8 (legible sin la passphrase, puede estar vacía)
—       48          texto cifrado (semilla cifrada de 32 bytes) || tag Poly1305 (16 bytes)
```

Los metadatos `public_key`/`created_at`/`label` son públicos:
legibles **sin la passphrase** (`public_metadata_from_bytes`). La
clave pública es pública por naturaleza; la etiqueta es solo un
nombre de visualización — ocultarla no aporta nada y cuesta el
listado sin passphrase. La longitud total depende de la etiqueta:
141 bytes con etiqueta vacía (KEYFILE_LEN + 32 + 8 + 1), más la
longitud de la etiqueta.

**v1 (obsoleta, exactamente 100 bytes)**: magic ‖ version=1 ‖ m ‖
t ‖ salt ‖ nonce ‖ texto cifrado — la clave pública NO se guardaba
(un archivo bloqueado era anónimo). Los archivos v1 siguen siendo
**legibles** (metadatos `None`, passphrase obligatoria para
`identity show`; el re-sellado vía `scone identity restore` los
migra a v2).

### Decisiones de diseño

- **Parámetros del KDF en el archivo**: `m_cost_mib` y `t_cost` se
  leen de la cabecera (no hardcodeados en lectura), lo que permite
  subirlos en versiones futuras sin romper los archivos existentes.
- **Cotas de seguridad en lectura**: un archivo que declare
  `m_cost_mib` > 64 o `t_cost` = 0 o > 10 se rechaza
  (`Error::InvalidParams`) *antes* de cualquier asignación — un
  archivo malicioso no puede obligar a un nodo a asignar gigas ni
  buclear indefinidamente. El valor por defecto en escritura es
  `m_cost_mib` = 64, `t_cost` = 3.
- **Magic + versión explícitos**: un archivo que no sea un archivo
  de claves (`Error::InvalidFile`) o de una versión no soportada
  (`Error::UnsupportedVersion`) se rechaza limpiamente, sin panic.
- **Passphrase errónea**: el fallo de autenticación Poly1305 no
  indica *por qué* falla; una passphrase errónea y un texto cifrado
  corrupto son pues indistinguibles y afloran como el mismo error
  distinto `Error::WrongPassphrase`.

## Derivación de la clave de cifrado

```text
key = Argon2id(passphrase, salt, m_cost = m_cost_mib × 1024 KiB,
               t_cost = t_cost, p_cost = 1, output = 32 bytes)
```

Luego:

```text
ciphertext || tag = XChaCha20-Poly1305-Seal(key, nonce, plaintext = seed)
```

Sin datos asociados (AAD) en v1. El nonce de 24 bytes, aleatorio por
archivo, hace despreciable el riesgo de reutilización (los archivos
de claves se escriben una sola vez).

## API (`scone-keystore`)

| Función | Rol |
|---|---|
| `create(path, passphrase) -> GeneratedKey` | Genera una clave, escribe el archivo cifrado (crea los directorios padre); **se niega a sobrescribir** un archivo existente (`Error::KeyfileExists`); devuelve la clave en memoria |
| `create_overwriting(path, passphrase) -> GeneratedKey` | Igual, pero reemplaza explícitamente un archivo existente |
| `open(path, passphrase) -> SigningKey` | Descifra y devuelve la clave de firma |
| `list(dir) -> Vec<KeyEntry>` | Lista los archivos `*.sconekey` de un directorio (ordenados por nombre); directorio ausente = lista vacía |

`GeneratedKey.signing_key` se pone a cero al hacer drop por
`ed25519-dalek` (feature `zeroize` por defecto). La clave AEAD
derivada, la semilla y el texto plano descifrado van envueltos en
`zeroize::Zeroizing`: el Drop cubre todas las rutas de salida,
incluidos los errores propagados con `?`.

## Permisos de archivo

En Unix, el archivo de claves se crea con permisos `0600`
(lectura/escritura solo del propietario), vía `OpenOptions` +
`set_permissions` — independientemente del umask del proceso. La
comprobación de existencia y la creación son atómicas
(`create_new`, `O_EXCL`): dos generaciones concurrentes no pueden
sobrescribirse silenciosamente.

## CLI

```bash
scone identity generate <n> [--dir <directorio>] [--passphrase-env VAR] [--force]
scone identity list [--dir <directorio>]
scone identity show <n> [--dir <directorio>] [--passphrase-env VAR]
```

- Directorio por defecto: `$HOME/.scone/keys`.
- El nombre de la identidad se valida estrictamente: vacío, `.`,
  `..`, `/`, `\\\\`, NUL, punto inicial y el sufijo `.sconekey` se
  rechazan (el nombre se convierte en nombre de archivo dentro del
  directorio de claves — ningún componente de ruta puede escapar
  del keystore).
- Sin `--passphrase-env`, la passphrase se pide en la terminal
  **con entrada enmascarada** (`rpassword`: sin eco; dos veces con
  confirmación para `generate`).
- `identity generate` **muestra la frase de recuperación BIP39 (24
  palabras)** exactamente una vez — entropía fresca de 256 bits,
  las palabras solas regeneran la misma clave (`scone identity
  restore`); nunca se guardan en ningún sitio.
- `identity generate` **se niega a sobrescribir** un archivo de
  claves existente; `--force` lo reemplaza explícitamente (nueva
  salt/nonce, nueva clave, nueva frase).
- `identity list` muestra, para los archivos v2, la clave pública y
  la etiqueta **sin la passphrase** (metadatos públicos).
- `identity show` lee los metadatos públicos del archivo v2 y
  muestra la clave pública, el `OwnerId` en hex minúsculas (64
  caracteres, misma convención que `DomainId`), la etiqueta y la
  fecha de creación **sin la passphrase**; un archivo v1 (sin
  metadatos) la sigue pidiendo.

## Lo que este formato deliberadamente no hace

- Sin claves múltiples por archivo (una semilla por archivo).
- Sin cambio de passphrase (si algún día llega: reescribir el
  archivo con una nueva salt/nonce).


## Identidad por defecto (cli-simplify-v1)

Los comandos que necesitan una identidad de firma (`domain
register`, `domain set`, `domain update`, `domain renew`) usan la
identidad ÚNICA del keystore sin `--identity` cuando hay
exactamente una. Con varias identidades el comando falla y las
lista (`scone identity list`); pasa `--identity <nombre>`
explícitamente.
