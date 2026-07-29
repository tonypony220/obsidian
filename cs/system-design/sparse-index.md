---
title: Sparse index — индекс по блокам, не по элементам
tags: [concept, data-structure, index]
date: 2026-06-22
---

# Sparse index

moc: [[system-design-moc]]
back: [[lsm-tree]]
next:
- [[bloom-filter]]

---

```
Sorted data (на диске):
┌──────────────────────────────────────┐
│ block 0: alice, bob, charlie         │ offset 0
├──────────────────────────────────────┤
│ block 1: dave, eve, frank            │ offset 4096
├──────────────────────────────────────┤
│ block 2: gina, harry, ian            │ offset 8192
├──────────────────────────────────────┤
│ block 3: jane, kate, leo             │ offset 12288
└──────────────────────────────────────┘

Sparse index (первый ключ каждого блока):
┌──────────────────┐
│ alice → 0        │
│ dave  → 4096     │   ← целиком в RAM, маленький
│ gina  → 8192     │
│ jane  → 12288    │
└──────────────────┘

lookup "eve":
  бин. поиск по sparse → eve >= dave & eve < gina → block 1
  read block 1 с диска (1 disk read, ~4 KB)
  бин. поиск внутри блока → eve найден
```

**TL;DR:** индекс, который хранит указатель не на каждый элемент, а только на **начало блока**. Полный (dense) индекс был бы как сами данные. Sparse — на порядки меньше, помещается в RAM целиком. Цена: после нахождения блока нужен один disk read, не точное попадание в конкретный элемент.

## Dense vs sparse

| | Dense index | Sparse index |
|---|---|---|
| Указатель на | каждый элемент | начало каждого блока |
| Размер | ~ как данные | данные / block_size |
| Лукап | 1 индекс read → 1 data read | 1 индекс read → 1 block read → поиск внутри |
| Где живёт | обычно на диске | в RAM |

При block size 4 KB и средней записи 100 байт — dense индекс был бы в **40 раз больше** sparse. На 1 TB данных:
- Dense ≈ 50 GB (не помещается в RAM).
- Sparse ≈ 1.2 GB (помещается).

## Как работает лукап

```python
def lookup(key, sparse_index, file):
    # 1. бинарный поиск по sparse: какой блок?
    i = bisect_right(sparse_index_keys, key) - 1
    if i < 0:
        return None
    block_offset = sparse_index[sparse_index_keys[i]]

    # 2. читаем блок с диска
    block = read_block(file, block_offset)

    # 3. бинарный поиск внутри блока в RAM
    return find_in_block(block, key)
```

Всего **один disk read** на лукап. Sparse index уже в RAM.

## Почему именно блоками 4–64 KB

Sweet spot между двумя крайностями:

- **Слишком маленький блок** (например, 128 байт) → sparse index почти такой же большой как данные. Теряется смысл sparse.
- **Слишком большой блок** (например, 16 MB) → каждый лукап читает 16 MB с диска ради 100 байт. Read amplification растёт.

4–64 KB — компромисс:
- Filesystem block = 4 KB (минимальная единица I/O в любом случае).
- Sequential read блока ≤ 64 KB занимает миллисекунды на NVMe.
- Sparse index получается ~1/100–1/1000 размера данных → помещается в RAM.

## Где используется

| Система | Как использует |
|---|---|
| **LSM SSTable** | sparse index в footer файла → найти data block по ключу |
| **Postgres BRIN** | min/max каждых N блоков heap — sparse range index |
| **Parquet, ORC** (колоночные форматы) | row group statistics: min/max, null count per group |
| **ClickHouse MergeTree** | primary key index — sparse, одна запись на granule (по умолчанию каждые 8192 строки) |
| **Filesystem inode** | extent tree — указатели на диапазоны блоков, а не на каждый block |
| **Virtual memory** | page table — sparse mapping virtual addresses → physical pages |

## ClickHouse как яркий пример

ClickHouse — OLAP-БД, **specifically** проектировалась под sparse index. Primary key индекс в ClickHouse — не B-tree, а массив значений primary key каждой 8192-й строки:

```
data:           [row 0..8191][row 8192..16383][row 16384..24575] ...

primary index:  pk[0] → block 0
                pk[8192] → block 1
                pk[16384] → block 2
                ...
```

На таблице в 1 миллиард строк primary index = 122 тысячи записей — помещается в RAM целиком, занимает несколько MB. Лукап = бинарный поиск в RAM → один read блока в 8192 строки → поиск внутри.

Это работает на ClickHouse потому, что данные **insert-only и физически отсортированы**. На update-heavy workload sparse index не подходит — нужен dense.

## Не только в БД

Sparse index — пример общего паттерна **иерархического индекса**: «грубый указатель в RAM + точный поиск в подгруженном куске». Тот же приём:

- **Page tables в OS**: virtual address делится на (page_directory, page_table, offset). Не один большой mapping, а несколько уровней — каждый sparse.
- **DNS hierarchical lookup**: root → TLD → authoritative server. Каждый уровень знает только «куда дальше», не финальный ответ.
- **CDN edge cache** + origin: edge как sparse index на популярные ресурсы, origin для всего остального.
- **HTTP range requests**: клиент знает примерное смещение нужного куска в большом файле — запрашивает только byte range, не весь файл.
- **CPU cache hierarchy**: L1 → L2 → L3 → RAM. Каждый уровень — sparse cache над следующим.

**Общее везде**: данных слишком много, чтобы держать точный индекс в быстром слое. Решение — **иерархия из грубого индекса (быстрый слой) и точного поиска (медленный слой, подгружается по необходимости)**. Это классический memory hierarchy trade-off: размер vs скорость доступа. Sparse index — конкретная реализация этого паттерна для отсортированных данных на диске.
