# Архитектура

## Сначала простыми словами

Пяти концепций достаточно, чтобы понять всё остальное:

1. **Имя** — адрес в интернете, например `mysite.scone`.
2. **Ваша фраза из 24 слов** — ваша личность. Это документ,
   подтверждающий, что имя принадлежит вам, и подписывающий каждое
   ваше изменение.
3. **Публичная книга записей** (блокчейн) — общий, публичный реестр
   того, что кому принадлежит. Никто не контролирует её, никто
   не может переписать её.
4. **Майнинг** — способ зарезервировать имя: ваш компьютер выполняет
   некоторую работу, одинаковую и справедливую для всех. Никаких
   очередей, никаких посредников.
5. **Реле** — компьютер, который поддерживает работу сети, храня
   копию книги и отвечая на вопросы. Любой может запустить реле;
   чем их больше, тем сильнее сеть.

Остальная часть страницы идёт глубже — вы можете остановиться здесь
и уже понимать проект.

## Руководящий принцип

Scone разделяет полномочия и доступность:

- **блокчейн** — источник истины о владении и порядке изменений;
- **DHT** — система дистрибуции/репликации, она **не** является
  источником полномочий;
- **локальное хранилище** содержит состояние узла, индексы и кэши.

Узел обязан рассматривать любые данные из DHT как потенциально
вредоносные: они принимаются только после криптографической проверки
(подпись владельца, соответствие владению / последовательности /
record_hash цепочки).

## Разделение ответственности

```text
BLOCKCHAIN
──────────
Ownership
Registration
Authorization
Ordering
Sequence
Record hash
Consensus information

DHT
───
DNS records
Signed records
Replication
Availability

LOCAL STORAGE
─────────────
Local state
Indexes
Cached DHT records
Blockchain state
```

## Крейты и зависимости

```text
scone-crypto     ← lowest layer: pure primitives over bytes
    ▲
scone-core       ← protocol types (depends on scone-crypto)
    ▲
scone-protocol  ← wire/blockchain binary format
    ▲
scone-blockchain← blockchain logic, canonical state (in memory)
    ▲               └ scone-topology (pure engine, zero network/fs/
                     async; fed by scone-network) — scone-genesis
                     (one-shot mainnet genesis tool binary)
scone-storage   ← local persistence: redb backend + verified snapshots
    ▲
scone-network   ← p2p relay: libp2p/QUIC, mempool, sync, DHT, DNS
    ▲
scone-keystore  ← encrypted `.sconekey` keyfiles (depends on scone-crypto)
    ▲
scone           ← binary / application
```

### Решение: `scone-core` зависит от `scone-crypto`

`DomainId` должен быть вычислим от самого основания
(`BLAKE3-256(prefix || canonical name)`), но конкретная криптография
не должна загрязнять ядро. Решение:

- `scone-crypto` — самый нижний слой: чисто детерминированные функции
  над байтами (плюс OS RNG для генерации ключей), нуль типов ядра,
  никаких зависимостей от сети/файловой системы/async → циклическая
  зависимость невозможна;
- `scone-core` использует его только для хеширования;
- подписи и ключи Ed25519 живут в `scone-crypto` и работают
  с байтами (канонические полезные нагрузки формируются
  `scone-protocol`).

### Роль каждого крейта

**scone-core** — имена (`TldName`, `DomainName`), идентификаторы
(`DomainId`, `OwnerId`, `PublicKeyRef`), DNS-данные (`DnsRecord`,
`SignedDnsRecord`), транзакции (`RegisterDomain`, `UpdateDomain`).
Детерминированный и переносимый. Запрещено в этом крейте: libp2p,
redb, tokio/async, сеть, файлы, ОС, DNS-сервер, логика блокчейна.

**scone-crypto** — криптографические примитивы. Сегодня: `hash256`
(BLAKE3), подписи и ключи Ed25519 (`SigningKey`,
`PublicKey`, `Signature`, строгая пакетная проверка), AEAD-конверт
групп записей (XChaCha20-Poly1305). Запланировано: дерево Меркла.
Правило: никаких самодельных алгоритмов, только проверенные
зависимости.

**scone-protocol** — бинарный формат протокола: каноническое
кодирование/декодирование (минимальный варинт LEB128, «голые»
32-байтовые идентификаторы) транзакций, DNS-записей, блоков
(`Block`/`BlockHeader` с непрозрачным полем консенсуса) и P2P-сообщений,
с проверкой ограничений размера до аллокации, отклонением
неканонических форм и версионированием (`PROTOCOL_VERSION`).
Никакой логики консенсуса, сети или хранилища.
Справочник по формату: `/docs/en/technical/protocol.md`.

**scone-blockchain** — логика блокчейна и каноническое состояние:
идентификация транзакций (`TxId`), упорядоченное дерево Меркла,
`BlockHash`, детерминированный генезис, валидация блоков и транзакций,
правила применения состояния (`DomainState`/`ChainState`, в памяти,
прямой доступ по `DomainId`), обнаружение форков и выбор ветви
(`fork_choice`: порог финальности + самая длинная цепочка +
разрешение ничьей по наименьшему хешу), финальность чекпоинтов
и абстракция консенсуса (`trait Consensus`, по умолчанию
разрешающая реализация). PoW регистрации (майнинг/проверка,
сложности для каждой сети) находится в `scone-core`. Запрещено
в этом крейте: сеть, DHT, хранилище (redb), DNS-сервер. Справка:
`/docs/en/technical/blockchain.md`.

**scone-storage** — локальное хранилище (поставляется). Конкретный
бэкенд — `redb` (`NodeStore`), скрытый за минимальным трейтом, так
что блокчейн/DHT/DNS никогда не связаны с redb напрямую; можно
подключить другой бэкенд (распределённый KV). Включает
пагинированный KV-бэкенд состояния, журнал блоков, проверенные
снапшоты (хешированный манифест + страницы) и путь восстановления/
повтора при перезапуске. Справка: `/docs/en/technical/storage.md`.

**scone-network** — p2p-реле узла: рой libp2p/QUIC, реле транзакций
и блоков (систематическая локальная валидация), ограниченный мемпул,
ограниченная синхронизация блоков (128 блоков/запрос), DHT Kademlia
для `SignedDnsRecord`, DNS-сервер UDP+TCP (GeoDNS всегда включён —
встроенная таблица GeoIP, рекурсивный фолбэк), цикл
чекпоинтов/пульсов якорей (CLI говорит напрямую по p2p QUIC
с реле, локального RPC больше нет). Достижимость и обнаружение:
AutoNAT v2 (вердикт reachable/NATed), circuit-relay v2 + DCUtR для
узлов за NAT, mDNS (обнаружение в локальной сети без конфигурации,
ВКЛЮЧЁН по умолчанию на devnet) и UPnP (проброска QUIC-порта
маршрутизатором, опционально). Конфигурация через файл `.env`
(см. `.env.example`). Справка: `/docs/en/technical/relay.md`.

**scone-keystore** — зашифрованное хранение ключей подписи Ed25519
(файлы `.sconekey`: Argon2id + XChaCha20-Poly1305). Этот крейт владеет
файловой системой, чтобы `scone-crypto` мог оставаться чистым.
Справка: `/docs/en/technical/keystore.md`.

**scone-topology** — ЧИСТЫЙ движок локальной оптимизации p2p-топологии:
нуль сети, нуль файловой системы, нуль async. Он не знает ни libp2p,
ни рой: `scone-network` передаёт ему наблюдения (RTT пинга, соединения)
и получает решения по управлению окружением.

**scone-genesis** — одноразовый инструмент (kickstarter) для подготовки
генезиса основной сети: версионируемый JSON-манифест, распределения,
детерминированный блок.

**scone** — бинарник/CLI. Зависит от крейта только тогда, когда
реально его использует (сегодня: scone-network для реле,
scone-blockchain, scone-storage, scone-keystore, scone-protocol,
scone-core, scone-crypto, scone-topology).

## Узел

Узел Scone объединяет:

1. блокчейн (валидация консенсуса, состояние владения);
2. DHT-маршрутизацию (публикация/получение `SignedDnsRecord`);
3. локальное хранилище (состояние, индексы, кэши);
4. резолвер / DNS-сервер на переднем крае.

### Запуск нового узла (бутстрап состояния, P0.3/P0.4)

Новый узел НЕ воспроизводит генезис: он стартует от проверенного
финализированного состояния.

```text
finalized checkpoint (committee, signature quorum — validated by
        │              the chain layer before any use)
        ▼
verified snapshot (manifest + paginated pages; the importer
        │              RECOMPUTES the SMT root from the entries
        │              and compares it to the signed state_root —
        │              a falsified/incomplete snapshot is rejected
        │              before any write)
        ▼
anchor block H (requested from the network BY HASH)
        │
        ▼
recent blocks H+1..H+k (relay, increasing height)
        │
        ▼
synced — at boot, replay of ONLY the suffix above H
```

Два профиля узла:

- **архивный узел** (флаг оператора): хранит все исторические блоки,
  отдаёт их сети, стартует через существующие пути;
- **узел с полным состоянием**: держит полное состояние на H и суффикс,
  но не историю ниже H — он валидирует будущие блоки и отдаёт
  состояние, но не отдаёт историю.

Этот путь OPT-IN: импорт проверенного снапшота — явное действие
оператора (`import_verified_snapshot`), никогда автоматическое —
поведение по умолчанию существующего узла не меняется.
Нормативные детали: `/docs/en/technical/storage.md`, разделы
«VERIFIED state snapshot» и «State bootstrap».

## Поток разрешения DNS

```text
DNS query (example.uip)
    │
    ▼
Node: DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || "example.uip")
    │
    ├── authoritative DNS snapshot (RAM) ─► unknown/expired → NXDOMAIN
    │                                          (without touching the core)
    ├── sharded DNS cache (qname,qtype) ───► hit: shared entry,
    │                                          lazy verification
    ├── local cache (SignedDnsRecord) ─────► verification (signature, owner,
    │                                          sequence, record_hash vs chain)
    ├── DHT get(DomainId) ─────────────────► same verification
    │                                          (miss → 30 s negative cache)
    ▼
DNS answer (if and only if valid)
```

DNS-снапшот (существование + срок действия каждого живого домена,
публикуется после каждого применённого блока) гарантирует, что
решение о существовании никогда не проходит через цикл событий реле:
ядро затрагивается только для фактического получения записей.

## Эволюционность

- версионируемые префиксы (`SCONE-DOMAIN-V1`, `SCONE-PUBKEY-V1`,
  `SCONE-OWNER-V1`, …): изменение правила вычисления создаёт новое
  пространство имён, а не ломает существующее;
- `RecordData` расширяется через варианты + `Unknown { type_code, data }`
  для типов, неизвестных на момент чтения;
- `PROTOCOL_VERSION` (`scone-protocol`) для ломающих изменений
  формата передачи;
- цепочка хранит только хеши: крупные данные остаются в DHT.
