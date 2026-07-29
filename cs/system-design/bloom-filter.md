---
title: Bloom filter — probabilistic membership test
tags: [concept, data-structure, probabilistic]
date: 2026-06-22
---

# Bloom filter

moc: [[system-design-moc]]
back: [[lsm-tree]]
next:
- [[sparse-index]]

---

```
bit array (m бит, изначально нули):
[0 0 0 0 0 0 0 0 0 0 0 0]
 0 1 2 3 4 5 6 7 8 9 10 11

insert("alice"):
  hash1("alice") % m = 3  → set bit 3
  hash2("alice") % m = 7  → set bit 7
  hash3("alice") % m = 1  → set bit 1

  [0 1 0 1 0 0 0 1 0 0 0 0]

contains("bob"):
  hash1("bob") % m = 5   → bit 5 = 0  → return NO (гарантированно)

contains("alice"):
  hash1, hash2, hash3 → bits 3, 7, 1  → все = 1  → return MAYBE

contains("eve"):
  hash1("eve") % m = 1   → bit 1 = 1 (set от alice)
  hash2("eve") % m = 3   → bit 3 = 1 (set от alice)
  hash3("eve") % m = 7   → bit 7 = 1 (set от alice)
                          → return MAYBE  ← false positive!
                            (eve никогда не добавлялась)
```

**TL;DR:** probabilistic structure для проверки «возможно есть» vs «точно нет». Битовый массив + k hash-функций. Insert: выставить k бит. Contains: все биты = 1 → MAYBE, любой 0 → NO. **False negatives невозможны**, false positives настраиваются размером (m) и числом hash'ей (k). Типично: 10 бит/элемент = 1% FP.

## Что отвечает bloom

| Ответ | Значит |
|---|---|
| **NO** | ключа в множестве **точно нет** (100%) |
| **MAYBE** | ключ **может быть** или **может не быть** (с вероятностью FP) |

Асимметрия: NO — детерминированный, MAYBE — вероятностный.

## Insert и contains

```python
class BloomFilter:
    def __init__(self, m, k):
        self.bits = [0] * m
        self.k = k

    def add(self, key):
        for i in range(self.k):
            idx = hash_i(key, i) % len(self.bits)
            self.bits[idx] = 1

    def contains(self, key):
        for i in range(self.k):
            idx = hash_i(key, i) % len(self.bits)
            if self.bits[idx] == 0:
                return False        # NO — гарантированно
        return True                 # MAYBE — может быть FP
```

## Параметры и формулы

| Символ | Смысл |
|---|---|
| n | сколько элементов вставлено |
| m | размер битового массива |
| k | количество hash-функций |
| p | вероятность false positive |

Формула FP:
```
p ≈ (1 - e^(-kn/m))^k
```

Оптимальное k при фиксированных m и n:
```
k_opt = (m/n) × ln(2) ≈ 0.693 × (m/n)
```

Практический расчёт: «сколько бит на элемент для FP = X%»:

| Бит на элемент (m/n) | Оптимальное k | FP rate |
|---|---|---|
| 8 | 6 | ~2% |
| 10 | 7 | ~1% |
| 15 | 10 | ~0.1% |
| 20 | 14 | ~0.05% |
| 30 | 21 | ~0.0001% |

LSM-движки обычно держат 10 бит/ключ → 1% FP. На 10M ключей = 12.5 MB в RAM на bloom.

## Почему false negatives невозможны

Если ключ был вставлен — его k битов **выставлены в 1**. При contains мы проверяем те же k битов (детерминированный hash) — они все 1, ответ MAYBE.

Биты могут стать «случайно» 1 от других вставок (это даёт FP), но **никогда не возвращаются в 0**. Поэтому ключ, который был добавлен, никогда не «потеряется».

## Удаление невозможно

В classical bloom **нельзя** удалить элемент. Если просто обнулить биты — можно нарушить чужие записи (тот же бит мог быть выставлен другим ключом).

Решения:
- **Counting Bloom filter** — вместо бит хранит счётчики (обычно 4 бита). Insert: ++, delete: --. Тратит больше памяти.
- **Cuckoo filter** — альтернативная структура с поддержкой удаления.
- **Rebuild** — при сильном устаревании пересобрать filter с нуля.

В LSM эта проблема обходится: SSTable immutable, bloom filter каждого SSTable строится один раз при создании. Удаления для SSTable не существует — он либо есть, либо целиком удалён compaction'ом.

## Где используется

| Система | Зачем |
|---|---|
| **LSM SSTable** (RocksDB, Cassandra, LevelDB) | Skip SSTable без disk read, если ключа точно нет |
| **Chrome Safe Browsing** | Локальный фильтр URL: «эта страница точно безопасна» vs «нужно спросить сервер» |
| **Bitcoin SPV clients** | Запрос «есть ли транзакции, касающиеся моих адресов» без раскрытия конкретных адресов |
| **CDN cache** | «Точно ли нет в cache» — избегать ненужного origin-запроса |
| **Distributed cache (Memcached, Redis)** | Перед запросом проверить «возможно ли кэш-hit» |
| **Database joins** | Bloom join: фильтр по одной таблице ускоряет лукап в другой |
| **PostgreSQL** | bloom-extension для индексов по множеству колонок |

## Не только в БД

Bloom — пример общего паттерна **probabilistic data structure**: торгуем точность ответа на резкое снижение memory/compute. Тот же класс структур:

- **HyperLogLog** — приближённый count distinct (Redis `PFCOUNT`). 12 KB → точность ±1% на миллиардах элементов.
- **Count-Min Sketch** — приближённые частоты («сколько раз видел этот ключ»). Streaming аналитика.
- **MinHash / SimHash** — приближённая похожесть множеств / документов. Дедупликация в search engines.
- **Skip-list level structure** — рандомизированная структура с probabilistic balancing.

**Общее везде**: класс задач, где **точный ответ дорогой** (полный поиск, точный count), а **приближённый ответ + односторонняя гарантия** дёшев и достаточен. Bloom специфичен тем, что гарантирует **точность одного из двух ответов** (NO) и допускает погрешность только в другом (MAYBE) — это асимметричная гарантия, очень удобная для «дёшево отсеять явные no, дорого проверять остальные».
