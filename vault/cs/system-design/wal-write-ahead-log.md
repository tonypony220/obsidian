---
title: WAL — Write-Ahead Log
tags: [concept, storage, durability, recovery]
date: 2026-06-22
---

# WAL — Write-Ahead Log

moc: [[system-design-moc]]
back: [[fsync-and-durability]]
next:
- [[lsm-tree]]
- [[lsm-vs-btree]]

---

```
client write
     │
     ▼
┌──────────────────────┐
│ 1. append to WAL     │ ← sequential I/O
│    "intend: set X=42"│
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ 2. fsync WAL         │ ← durability barrier
└──────────┬───────────┘
           │ ack клиенту (можно)
           ▼
┌──────────────────────┐
│ 3. apply to data     │ ← page cache, eventually disk
│    (отложенно,       │   (или memtable для LSM)
│     батчево,         │
│     асинхронно)      │
└──────────────────────┘

crash:
  ▶ data on disk = неполное состояние
  ▶ WAL = source of truth о намерениях
  ▶ recovery: replay WAL с последнего checkpoint → consistent
```

**TL;DR:** WAL — append-only лог «намерений», записываемый **до** реального изменения данных. Гарантия: либо запись закоммичена и видна, либо crash отменит её при recovery — никогда не будет «полу-применённого» состояния. Sequential write вместо random, плюс возможность group commit'ить.

## Основная идея

**Write-ahead** = запиши намерение **раньше** чем действие.

Без WAL: меняем data page → crash посередине → страница в полу-обновлённом состоянии, восстановить нечем.

С WAL:
1. Append записи в WAL: «хочу заменить X=10 на X=42» (intent + before + after image).
2. fsync WAL.
3. Только теперь можно ack клиенту.
4. Реальное изменение data page — отложено, может быть батчевым.

При crash: data page может быть в любом состоянии — но WAL знает, что хотели сделать.

## Структура записи WAL

Каждый log record содержит:

- **LSN** (Log Sequence Number) — монотонно растущий ID записи.
- **Transaction ID** — к какой транзакции относится.
- **Operation type**: insert / update / delete / commit / abort / checkpoint.
- **Before image** (старое значение) — нужен для undo.
- **After image** (новое значение) — нужен для redo.
- **Checksum**.

Файл WAL — последовательность таких записей, append-only.

## Recovery — что делает БД при старте

ARIES-алгоритм (Postgres, MySQL InnoDB используют варианты):

1. **Analysis pass**: пройти WAL от последнего checkpoint. Восстановить список активных транзакций на момент crash.
2. **Redo pass**: применить **все** изменения из WAL начиная с checkpoint. Не важно, был ли commit — просто перепрогнать. Получается состояние «как было на момент crash».
3. **Undo pass**: для транзакций, которые не закоммитились до crash — откатить через before image. Состояние = «как будто их не было».

Результат: durable + consistent. Закоммиченные транзакции применены, незакоммиченные исчезли.

## Checkpoint — точка обрезки WAL

Без checkpoint WAL рос бы бесконечно, recovery становилась бы всё дольше.

Checkpoint = «гарантирую: все dirty pages с LSN ≤ X сброшены на disk».

Алгоритм:
1. Заморозить новые writes на секунду (или использовать fuzzy checkpoint без заморозки).
2. Сбросить dirty pages из page cache на disk через fsync.
3. Записать в WAL «checkpoint at LSN X».
4. WAL до LSN X можно удалять.

Postgres: автоматически каждые `checkpoint_timeout` или при заполнении `max_wal_size`.

## Почему это быстро

| Без WAL | С WAL |
|---|---|
| Random write per транзакция | Sequential append to WAL + отложенный random write |
| Каждый commit ждёт data flush | Каждый commit ждёт только WAL fsync |
| Нет батчинга — page = единица | Group commit: N транзакций в 1 fsync |

Главный выигрыш — **sequential I/O вместо random + амортизация fsync** (см. [[fsync-and-durability]]).

## Где используется

| Система | WAL называется |
|---|---|
| Postgres | `pg_wal/` (раньше `pg_xlog`) |
| MySQL InnoDB | redo log (`ib_logfile*`) + undo log |
| SQLite | WAL mode (`*-wal` файл) |
| LSM-БД (RocksDB, Cassandra) | WAL для memtable recovery |
| ext4, xfs, btrfs | journal |
| Kafka | сам топик **является** WAL'ом (consumers читают log) |
| Raft consensus | log entry перед commit |
| etcd, ZooKeeper | snapshot + WAL |

## WAL vs LSM

Интересная связь:
- В LSM-БД WAL нужен для **recovery memtable** (memtable в RAM, crash = потеря).
- Сам SSTable — append-only immutable, тоже «log-like».
- Можно сказать: LSM — это БД, где **весь storage** построен на принципе «append-only log of changes», а WAL — частный случай этого.

В B-tree БД WAL — отдельный слой над random-access storage. В LSM граница размывается.

## Подводные камни

- **WAL fsync — bottleneck коммитов**. Все TPS любой ACID-БД упираются сюда.
- **WAL на медленном диске убивает throughput**. Дешёвое решение в облаке: вынести WAL на отдельный быстрый volume.
- **Незакоммиченные транзакции тоже пишутся в WAL** — занимает место и I/O. Длинные транзакции = много WAL.
- **Replication**: реплики читают WAL мастера. Лаг репликации = лаг чтения и применения WAL.
- **WAL bloat**: replication slot, который не двигается (отвалившаяся реплика), не даёт обрезать WAL → диск заканчивается.

## Не только в БД

WAL — частный случай **event log как source of truth**: вместо «текущее состояние = ground truth» использовать «лог событий = ground truth, состояние — derived».

Тот же приём:

- **Event sourcing** (архитектура приложений): не хранить «текущее состояние заказа», а хранить лог `OrderCreated`, `OrderPaid`, `OrderShipped`. Текущее состояние — fold log'а. Любой момент в прошлом — replay до нужной точки.
- **Kafka**: топик буквально является WAL'ом. Consumer'ы реализуют разные derived state'ы поверх одного лога.
- **Git**: коммиты = WAL изменений файлов. Рабочая копия = текущий state, восстанавливается из лога.
- **Blockchain**: распределённый append-only лог транзакций; balance — derived.
- **CRDT и распределённые системы**: log of operations + merge function = eventual state.

**Общее везде**: **separation of intent and state**. Лог — durable, упорядоченный, immutable. Состояние — derived, восстанавливаемое, отбрасываемое. Это переворачивает обычную модель «state — ground truth» и даёт recovery, audit, time-travel, replication «бесплатно».
