---
title: LSM vs B-tree — как работают
tags: [concept, storage, database]
date: 2026-06-21
---

# LSM vs B-tree — как работают

moc: [[system-design-moc]]
back: [[lsm-tree]]
next:
- [[wal-write-ahead-log]]
- [[fsync-and-durability]]

---

```
B-tree (in-place update)            LSM (append-only)

      [root page]                   write → memtable (RAM)
     /     |    \                              ↓ flush when full
  [page] [page] [page]               ┌ SSTable₃ (newest, smallest)
   / \   / \   / \                   ├ SSTable₂
  leaf leaf leaf leaf                ├ SSTable₁
  (data updated                      └ SSTable₀ (oldest, largest)
   IN PLACE)
                                     compaction merges levels →
                                     drops old versions + tombstones

read:  O(log n) page reads          read:  memtable + bloom + N SSTables
write: O(log n), random I/O         write: O(1), sequential I/O
```

**TL;DR:** B-tree обновляет данные **на месте** в страничном дереве на диске (random I/O, стабильное чтение). LSM пишет **append-only** через memtable → immutable SSTable, объединяет фоновым compaction (sequential I/O, очень быстрая запись ценой чтения через несколько уровней).

## B-tree — механика

**Структура**: сбалансированное дерево из страниц фиксированного размера (обычно 8–16 KB). Внутренние страницы держат ключи + указатели на детей; leaves держат сами данные (или указатели на heap, как в Postgres).

**Write path**:
1. Спуститься по дереву от корня к leaf через бинарный поиск на каждой странице.
2. Прочитать leaf-страницу с диска (если не в page cache).
3. Записать запись WAL → fsync (durability).
4. Обновить страницу в page cache, пометить грязной.
5. Если страница переполнилась → **page split**: разделить пополам, ключ-разделитель протолкнуть наверх. Каскад до корня в худшем случае.
6. Грязные страницы сбрасываются на диск отложенно (checkpoint).

**Read path**: те же O(log n) лукапов. Верхние уровни почти всегда в page cache → реально 1–2 disk read'а на запрос.

**Где живёт**: Postgres, MySQL InnoDB, SQLite, BoltDB, LMDB, WiredTiger (B-tree mode), MongoDB (через WiredTiger).

**Цена**:
- Random I/O на write — каждая обновляемая страница в случайном месте диска.
- Page splits → дополнительные writes при росте.
- Hot pages переписываются N раз подряд.

## LSM — механика

**Структура — три слоя**:
- **Memtable** (RAM) — sorted-структура (skip-list, RB-tree). Принимает все writes.
- **WAL** — append-only лог на диске. До ack клиенту — fsync на WAL.
- **SSTable** (Sorted String Table) — immutable файл на диске: sorted key→value + sparse index + Bloom filter.

**Write path**:
1. Append записи в WAL → fsync.
2. Insert в memtable (in-memory sort).
3. Ack клиенту. Всё — O(1) дискового I/O.
4. Когда memtable достиг порога (например, 64 MB) → **flush**: записать его как новый SSTable, очистить memtable, обрезать WAL.

**Read path** (дороже):
1. Проверить memtable.
2. Идти по SSTable от нового к старому. Для каждого:
   - Спросить **Bloom filter**: «ключ точно отсутствует?». Если да — пропустить без disk read.
   - Если может быть — прочитать SSTable.
3. Если ключ найден в нескольких SSTable — берётся самая свежая версия (LSM хранит **все** версии до compaction).
4. **Tombstone** — маркер удаления, тоже считается версией.

**Compaction** — фоновый процесс:
- Берёт несколько SSTable, мерджит в один (или несколько) более крупных.
- Дедуплицирует ключи: оставляет только свежую версию.
- Удаляет tombstones (если старше всех активных snapshot'ов).
- Цель: сократить число SSTable'ов на read path.

Две стратегии compaction:
- **Size-tiered** (Cassandra default): мерджит SSTable одного размера. Меньше write amplification, больше disk usage (дубликаты между tier'ами).
- **Leveled** (RocksDB default): фиксированные уровни L0, L1, ..., каждый в N раз больше предыдущего. Меньше дубликатов, стабильнее read, выше write amplification.

## Amplification — главная метрика

Три вида «во сколько раз реальная работа больше юзер-операции»:

| Amplification | B-tree | LSM size-tiered | LSM leveled |
|---|---|---|---|
| Write (байт на диск / байт от юзера) | ~1–2× | ~3–5× | ~10–30× |
| Read (disk reads / запрос) | 1–2 | N SSTables (~3–5) | 1 на уровень (~5–7) |
| Space (место на диске / размер данных) | 1× | 1.5–2× | ~1.1× |

LSM торгует **write amp на read/space amp**. Выбор стратегии compaction = где сесть на Парето-кривой.

## Когда что

| Профиль | Выбор |
|---|---|
| OLTP смешанный, средние writes | B-tree |
| Write-heavy (логи, телеметрия, чат, события) | LSM |
| Read-mostly, редкие updates | B-tree |
| Append-only с TTL | LSM (целые SSTable дропаются) |
| Жёсткий p99 на чтение | B-tree |
| Embedded, маленький footprint | B-tree (BoltDB) или LSM (RocksDB/LevelDB) — зависит от workload |

Durability и почему `fsync` дорогой → [[fsync-and-durability]].
