---
title: fsync и durability — почему durable writes дорогие
tags: [concept, storage, os, durability]
date: 2026-06-21
---

# fsync и durability

moc: [[system-design-moc]]
back: [[lsm-vs-btree]]
next:
- [[write-throughput-vs-connection-count]]
- [[lsm-tree]]

---

```
write(fd, ...)
       │
       ▼
┌──────────────┐
│  page cache  │  ← RAM в ядре
│    (RAM)     │     write() вернулся, но данные ЗДЕСЬ
└──────┬───────┘
       │ kernel сбросит «когда-нибудь» (секунды)
       ▼
┌──────────────┐
│  block layer │
└──────┬───────┘
       ▼
┌──────────────┐
│ drive cache  │  ← cache контроллера диска
└──────┬───────┘
       ▼
┌──────────────┐
│ flash/platter│  ← физический носитель — durable
└──────────────┘

fsync(fd) ── блокируется до тех пор, пока данные не дойдут сюда ──┘
```

**TL;DR:** fsync = syscall «не возвращай управление, пока данные durably на физическом носителе». Без него `write()` кладёт байты в page cache ядра — выдернут питание, данные исчезли. Цена fsync = синхронное ожидание носителя: HDD ~5–10 ms, NVMe ~50–200 µs, enterprise NVMe с PLP ~10–50 µs. Это нижняя граница write throughput любой ACID-БД при коммите. Амортизируется group commit'ом.

## Что без fsync

- `write()` → байты в **page cache** ядра (RAM, не на диске).
- Ядро решит сбросить когда-нибудь (vm.dirty_writeback_centisecs, обычно 5 сек batch'ами).
- Power loss до flush → данные исчезли. Программа уже получила «ok» — не узнает.

## Что делает fsync

`fsync(fd)` — POSIX syscall. Блокируется до пути:
page cache → block layer → drive → flash cell / platter.

Цена зависит от носителя — это **физика**, не алгоритмика:
- HDD — дождаться оборота пластины (~6 ms на 10k rpm).
- SSD — дождаться программирования flash-страницы.
- Enterprise NVMe с PLP (power-loss protection) — конденсатор гарантирует сброс cache при power loss → fsync возвращается сразу после write в drive cache.

## Где обязателен

- **БД при коммите транзакции** → fsync на WAL. Без этого ломается **D** в ACID.
- **Файловая система journal** (ext4, xfs).
- **Message broker** с persistent гарантией (Kafka `acks=all`).
- **Distributed consensus** (Raft): пишет log entry → fsync → ack лидеру.
- Финансы, healthcare, любая система где «зафиксировано» ≠ «возможно зафиксировано».

## Цена в числах

| Носитель | fsync latency | Коммитов/сек (1 fsync на commit) |
|---|---|---|
| HDD 7200rpm | 5–10 ms | 100–200 |
| SATA SSD consumer | 0.5–1 ms | 1000–2000 |
| NVMe consumer | 50–200 µs | 5000–20000 |
| NVMe enterprise + PLP | 10–50 µs | 20000–100000 |

Цифры объясняют, почему «один Postgres на HDD держит ~150 TPS» — это не Postgres, это диск.

## Group commit — амортизация

Один fsync gates **все** writes, попавшие в WAL до него. Идея:

```
T1 commit ─┐
T2 commit ─┼─► WAL: append append append ─► один fsync ─► ack T1, T2, T3
T3 commit ─┘
```

N транзакций → 1 fsync → throughput = N × (1 / fsync_time).

Стратегии:
- **Timer-based**: подождать X мкс перед fsync, набрать пачку. Postgres: `commit_delay`, `commit_siblings`.
- **Pipeline**: пока предыдущий fsync в полёте, копим следующую пачку. Используется в WAL Kafka, RocksDB.
- **Линковка клиентов**: Kafka `linger.ms` — продюсер группирует записи перед отправкой.

Trade-off: latency vs throughput. Больше group → выше throughput, каждая транзакция чуть дольше ждёт.

## Семейство syscall'ов

| Что | Гарантирует |
|---|---|
| `write()` | байты в kernel page cache |
| `fsync(fd)` | данные + metadata на disk |
| `fdatasync(fd)` | данные на disk, metadata только если влияет на read (быстрее) |
| `open(O_SYNC)` | каждый `write()` неявно делает sync (медленно) |
| `open(O_DIRECT)` | bypass page cache; **не** про durability — про предсказуемость |
| `msync(addr, MS_SYNC)` | то же для mmap-региона |

БД обычно используют `fdatasync` для WAL — metadata WAL-файла редко меняется, fdatasync дешевле.

## Подводные камни

- **fsync может молча соврать**. Дешёвые SSD ack'ают до physical write. Известные кейсы потери данных у MongoDB/Postgres в первые годы SSD.
- **«fsync gate»** в ext4: до 2018 fsync молча терял ошибки записи в page cache (выбрасывал и забывал). Исправлено в ядре 4.16+.
- **Контейнеры**: fsync проходит через cgroup, потенциально throttled при I/O limits.
- **Виртуализация / cloud**: fsync на гостевой ОС не означает durable на bare metal. У AWS EBS своя модель durability — fsync доходит до replicated block storage, не до physical disk.

## Не только в БД

fsync — частный случай **durability barrier**: явная точка в коде, где асинхронная система обязана дать синхронную гарантию commit до ack. Тот же паттерн встречается на всех уровнях:

- **Message queues**: Kafka `acks=all` ждёт fsync на всех ISR-репликах перед ack продюсеру.
- **Файловые системы**: journal commit ставит barrier перед reuse блоков.
- **Distributed consensus**: Raft — log entry должен быть persistent на majority до commit.
- **CPU**: memory barriers (`mfence`, `dmb`) — «не продолжай пока writes до меня не видны другим ядрам». То же на уровне процессора.
- **Network**: TCP `flush` перед close — гарантирует, что отправленное дошло до peer.
- **GPU**: `glFlush` / `glFinish` — заставить драйвер дождаться выполнения команд.

**Общее везде**: установить **happens-before** через явный barrier. Без barrier система видит более слабую модель, чем интуитивно ожидается (eventual vs sequential). Это базовый приём для случая, когда асинхронная система должна выглядеть синхронной в конкретной точке — ценой ожидания.
