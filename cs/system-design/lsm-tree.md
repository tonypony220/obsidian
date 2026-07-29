---
title: LSM-tree — log-structured merge tree
tags: [concept, storage, database]
date: 2026-06-21
---

# LSM-tree

moc: [[system-design-moc]]
back: [[architecture-stack]]
next:
- [[lsm-vs-btree]]
- [[wal-write-ahead-log]]
- [[fsync-and-durability]]
- [[bloom-filter]]
- [[sparse-index]]
- [[write-throughput-vs-connection-count]]

---

```
write ──► WAL (append + fsync) ──► memtable (RAM, sorted)
                                         │ когда заполнился (~64 MB)
                                         ▼
                                  flush: одним sequential write
                                         │
                                         ▼
                              ┌────────────────────┐
                              │ SSTable_3 (новый)  │ ─┐
                              ├────────────────────┤  │
                              │ SSTable_2          │ ─┼─► compaction
                              ├────────────────────┤  │   (merge, drop
                              │ SSTable_1 (старый) │ ─┘    tombstones)
                              └────────────────────┘
```

**TL;DR:** LSM = log-structured merge tree, **движок хранения** на диске. Все writes append-only: memtable (RAM) → flush в immutable SSTable → фоновый compaction. Sequential I/O, очень быстрая запись ценой более дорогого чтения. Используется и в NoSQL (Cassandra, ScyllaDB), и в SQL (MyRocks, CockroachDB, TiDB).

## Основная идея в одной фразе

**Никогда не меняй данные на диске. Только дописывай новые файлы и удаляй старые целиком.**

Random write дорогой, sequential — дешёвый. LSM делает **только** sequential writes ценой более сложного чтения.

## Зачем

- Append-only = sequential disk I/O. NVMe выдаёт ~7 GB/s sequential vs ~500k IOPS random — порядки разницы.
- Memtable в RAM = write подтверждается за микросекунды. Durability держит WAL (см. [[fsync-and-durability]], [[wal-write-ahead-log]]).

## На пальцах — write

Мини-БД с тремя ключами: `alice`, `bob`, `charlie`. Пройдём шаг за шагом.

### 1. Старт — всё пусто

```
RAM:           ┌──────────┐
               │ memtable │ (пусто)
               └──────────┘
Диск (WAL):    (пусто)
Диск (SSTable):(пусто)
```

### 2. write alice=10

```
1. WAL: append "SET alice=10" + fsync.
2. memtable: alice → 10.
3. ack клиенту.
```

memtable:
```
  alice → 10
```

### 3. write bob=20

То же самое:
```
WAL:      [SET alice=10][SET bob=20]
memtable: alice → 10, bob → 20   (отсортировано)
```

### 4. write charlie=30 → memtable заполнился → FLUSH

Допустим лимит memtable = 3 записи (в реале 64 MB).

```
1. Текущий memtable замораживается (read-only).
2. Создаётся новый пустой memtable для следующих writes.
3. Фоновый поток пишет замороженный memtable как файл — SSTable_1.
4. После записи SSTable_1 WAL можно обрезать.
```

После flush:
```
RAM:          ┌──────────────┐
              │ memtable_new │ (пуст)
              └──────────────┘

Диск:         ┌─────────────────────┐
              │ SSTable_1           │  ← immutable
              │  alice   → 10       │     sorted
              │  bob     → 20       │
              │  charlie → 30       │
              └─────────────────────┘
```

### 5. write alice=99 — старая версия остаётся

```
memtable_new: alice → 99
```

**alice есть в двух местах**: 99 в memtable, 10 в SSTable_1. Старое значение никто не трогал.

### 6. Второй flush — две версии на диске

```
Диск:
┌─────────────────────┐  ┌─────────────────────┐
│ SSTable_1 (старый)  │  │ SSTable_2 (новый)   │
│  alice   → 10       │  │  alice   → 99       │ ← обе версии
│  bob     → 20       │  │  ...                │   живут
│  charlie → 30       │  │                     │
└─────────────────────┘  └─────────────────────┘
```

### 7. delete bob — пишется tombstone

Нельзя стереть запись из immutable SSTable. Поэтому delete = **write со специальным маркером**:

```
memtable: bob → TOMBSTONE
```

После flush:
```
┌─────────────┐  ┌─────────────┐  ┌──────────────┐
│ SSTable_1   │  │ SSTable_2   │  │ SSTable_3    │
│ alice → 10  │  │ alice → 99  │  │ bob → TOMB   │
│ bob   → 20  │  │             │  └──────────────┘
│ charlie→30  │  │             │
└─────────────┘  └─────────────┘
```

### 8. Compaction — фоновое слияние

Через какое-то время накопилось много SSTable. Фоновый процесс **сливает** их в один новый:

```
До:
┌─────────────┐  ┌─────────────┐  ┌──────────────┐
│ SSTable_1   │  │ SSTable_2   │  │ SSTable_3    │
│ alice → 10  │  │ alice → 99  │  │ bob → TOMB   │
│ bob   → 20  │  │             │  └──────────────┘
│ charlie→30  │  │             │
└─────────────┘  └─────────────┘

После:
┌──────────────────┐
│ SSTable_merged   │
│ alice   → 99     │ ← свежая версия
│ charlie → 30     │
└──────────────────┘
                     (bob исчез: tombstone применился и удалился)
```

Compaction делает одновременно:
- Дедуплицирует ключи (оставляет свежую версию).
- Применяет tombstone'ы.
- Сокращает число SSTable → быстрее read.

## На пальцах — read

### Где может лежать ключ

```
RAM:    active memtable
        frozen memtable    (если идёт flush)

Диск:   L0: [SST] [SST] [SST]     ← из flush'ей, могут пересекаться
        L1: [SST a..c] [SST d..f] ← НЕ пересекаются, в 10× больше L0
        L2: ...
        L3: ...
```

Один ключ может присутствовать в **нескольких местах одновременно** (старые версии).

### Правило: от нового к старому, первый hit = ответ

```
active memtable → frozen → L0 (новый→старый) → L1 → L2 → ... → LN
```

Как только нашли ключ — **сразу return**. Свежая версия побеждает. Если первый hit = tombstone → return null.

### Что внутри SSTable

Файл с отсортированными парами. Чтобы найти ключ без full scan, используются две вспомогательные структуры:

- **[[bloom-filter]]** — bit array + N hash-функций. Отвечает: «ключа точно нет» (NO) или «возможно есть» (MAYBE). В RAM. Если NO — пропускаем SSTable **без disk read**.
- **[[sparse-index]]** — «первый ключ каждого блока + offset». В RAM. Бинарный поиск → один disk read на нужный блок.

Итог: максимум **один disk read per SSTable**.

### Read path на примере

Состояние:
```
RAM:    active: { dave: 5 }
Диск:   L0 SSTable_3: { bob: 10, charlie: 20 }
        L0 SSTable_2: { alice: 50 }
        L1 SST(a..c): { alice: 100, bob: 5 }
        L1 SST(d..f): { dave: 1, eve: 50 }
```

**read("bob")**:
```
1. active: нет.
2. L0 SSTable_3: bloom → MAYBE → sparse → block → bob=10 ✓ return
```
Не идём в L1, хотя там есть `bob: 5` — старее.

**read("alice")**:
```
1. active: нет.
2. L0 SSTable_3: bloom → NO       ← skip без disk read
3. L0 SSTable_2: bloom → MAYBE → block → alice=50 ✓ return
```

**read("frank")** (ключа нет):
```
1. active: нет.
2. L0 SSTable_3: bloom → NO
3. L0 SSTable_2: bloom → NO
4. L1 SST(a..c): диапазон не покрывает frank — skip.
5. L1 SST(d..f): bloom → NO
6. Нет L2 → return null
```

**0 disk reads** для отсутствующего ключа — bloom отсёк всё.

**read("dave")**:
```
1. active: dave=5 ✓ return
```
До диска не дошли.

### ASCII схема read

```
read(key)
   │
   ▼
┌────────────────────┐
│ active memtable    │── HIT → return
└─────────┬──────────┘
          │ miss
          ▼
┌────────────────────┐
│ frozen memtable    │── HIT → return
└─────────┬──────────┘
          │ miss
          ▼
┌────────────────────────────────────────┐
│ L0: для каждого SSTable (новый→старый) │
│   bloom(key)?                          │
│   ├─ NO    → skip (0 disk reads)       │
│   └─ MAYBE:                            │
│       sparse → block offset            │
│       read 1 block (1 disk read)       │
│       найти ключ → HIT → return        │
└─────────┬──────────────────────────────┘
          │ miss
          ▼
┌────────────────────────────────────────┐
│ L1: найти SSTable где диапазон ⊃ key   │
│     (≤ 1 SSTable, диапазоны не         │
│      пересекаются)                     │
│   bloom → sparse → block → HIT/miss    │
└─────────┬──────────────────────────────┘
          │ miss
          ▼
   ... L2, L3, ..., LN
   нигде нет → return null
```

### Range read (scan) — отдельный случай

`SELECT * WHERE key BETWEEN 'a' AND 'c'` — точечный bloom **не помогает** (он отвечает только про конкретный ключ).

Range read открывает **итераторы** во всех источниках, покрывающих диапазон (memtable, frozen, все L0, по 1 на L1+), затем merge-итератор: на каждом шаге минимальный текущий ключ, при дубликатах — свежая версия.

Дороже точечного read. Поэтому **leveled compaction** лучше для range — на каждом уровне ≤ 1 SSTable вместо N в L0.

### Read amplification

Сколько disk reads на 1 запрос:

| | B-tree | LSM (leveled) |
|---|---|---|
| Точечный read существующего ключа | 1–2 | до 5–10 |
| Точечный read отсутствующего ключа | 1–2 | 0 (благодаря bloom) |
| Range scan | log(N) + диапазон | merge всех источников |

LSM хуже на точечном read для tail-случаев (ключ на глубоком уровне) — отсюда худшие p99. Но за счёт bloom средняя нагрузка не катастрофична.

## Где используется

| Продукт | Модель данных | LSM-движок |
|---|---|---|
| Cassandra, ScyllaDB | wide-column / KV | свой |
| RocksDB, LevelDB | embedded KV | свой (LevelDB → RocksDB форк) |
| MyRocks | MySQL (relational SQL) | RocksDB |
| CockroachDB | distributed SQL (Postgres wire) | Pebble (форк RocksDB) |
| TiDB | distributed SQL (MySQL wire) | TiKV / RocksDB |
| HBase | wide-column | свой (Bigtable-наследник) |
| InfluxDB | time-series | свой |

### Индексы в LSM-БД с SQL — как лежат физически

LSM **не запрещает** secondary индексы. В SQL-БД на LSM (MyRocks, CockroachDB, TiDB) индексы реализованы через **разные ключевые префиксы в одном LSM-keyspace**. Физически — один storage engine, один WAL, одни SSTable. Логически — каждый префикс ведёт себя как отдельный индекс.

Пример: таблица `users` с primary key `id`, secondary index по `email` и GIN-like по `tags`.

```sql
CREATE TABLE users (id BIGINT PK, email TEXT UNIQUE, name TEXT, tags TEXT[]);
CREATE INDEX idx_users_tags ON users USING GIN(tags);

INSERT users VALUES (1, 'alice@a.com', 'Alice', ['admin','beta']);
INSERT users VALUES (2, 'bob@b.com',   'Bob',   ['beta']);
INSERT users VALUES (3, 'eve@e.com',   'Eve',   ['admin']);
```

**В Postgres** — разные файлы:

```
base/<db_oid>/
  16384       heap (3 строки)
  16385       B-tree PK (id → TID)
  16386       B-tree email (email → TID)
  16387       GIN tags (tag → posting list of TIDs)
```

**В CockroachDB** — один LSM-keyspace, разные префиксы:

```
/users/pk/1                       → ('alice@a.com', 'Alice', ['admin','beta'])
/users/pk/2                       → ('bob@b.com',   'Bob',   ['beta'])
/users/pk/3                       → ('eve@e.com',   'Eve',   ['admin'])

/users/idx_email/alice@a.com      → 1            ← значение = PK
/users/idx_email/bob@b.com        → 2
/users/idx_email/eve@e.com        → 3

/users/idx_tags/admin/1           → ∅            ← PK в ключе
/users/idx_tags/admin/3           → ∅
/users/idx_tags/beta/1            → ∅
/users/idx_tags/beta/2            → ∅
```

Все ключи **отсортированы вместе** в одном LSM-tree. SSTable содержит primary + secondary вперемешку. SQL-движок над Pebble/RocksDB знает, какой префикс — какой индекс.

`SELECT * FROM users WHERE email = 'bob@b.com'`:
1. Range scan по префиксу `/users/idx_email/` → `bob@b.com` → значение `2`.
2. Point lookup по `/users/pk/2` → строка Bob.

Два LSM-лукапа — то же, что Postgres делает с двумя B-tree файлами. Концептуально идентично, физически — один engine с разделёнными namespace'ами.

Ограничения индексов в Cassandra/ScyllaDB — следствие **распределённой шардированной модели** (scatter-gather по всем шардам), а не LSM как структуры. Решение там — materialized view (денормализация).

## Trade-off vs B-tree

| Метрика | B-tree | LSM |
|---|---|---|
| Write throughput | средний | очень высокий |
| Read latency | низкая, стабильная | выше, варьируется |
| Disk usage | компактнее | оверхед от compaction |
| Write amplification | низкая | средняя—высокая |
| Update | in-place | append + tombstone |

Подробно — [[lsm-vs-btree]].

## Когда выбирать

- Write-heavy: логи, метрики, события, чат, IoT-телеметрия.
- Append-mostly с редкими updates.
- TTL-данные: старые SSTable дропаются compaction'ом целиком.

Когда **не** выбирать:
- Read-heavy с жёстким p99 → стабильность B-tree важнее.
- Малая нагрузка → накладные расходы LSM не окупаются.

## Не только в NoSQL

«LSM = NoSQL» — историческая корреляция, не правило. Storage engine ортогонален модели данных. Три независимые оси системы хранения:

- **Модель данных**: relational / KV / document / graph / time-series — как выглядят запросы.
- **Движок хранения**: B-tree / LSM / column store — как байты живут на диске.
- **Распределение**: single-node / replicated / sharded — как масштабируется.

Любая комбинация существует в проде:
- SQL + B-tree + single-node → Postgres.
- SQL + LSM + distributed → CockroachDB.
- KV + LSM + distributed → Cassandra.
- KV + B-tree + embedded → BoltDB.
- Document + B-tree + sharded → MongoDB (WiredTiger).

Историческая привязка: NoSQL и LSM пришли одной волной (Bigtable 2006 → Cassandra/HBase 2008). Это два независимых ответа на write-heavy на масштабе: NoSQL отказался от joins ради шардирования, LSM отказался от in-place update ради sequential I/O. Связаны историей, не природой.

**Общее везде**: разделение «модель / движок / распределение» — пример декомпозиции системы на **ортогональные оси**. Тот же приём: [[control-plane-data-plane]] (управление / трафик), [[hex-architecture]] (домен / IO), [[edge-vs-serverless]] (логика / расположение). Инвариант: оси можно крутить независимо, на пересечениях получаются конкретные продукты.
