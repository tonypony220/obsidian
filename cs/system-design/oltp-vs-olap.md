---
title: OLTP vs OLAP — операционные транзакции vs аналитика
tags: [concept, database, architecture]
date: 2026-06-22
---

# OLTP vs OLAP

moc: [[system-design-moc]]
back: [[architecture-stack]]
next:
- [[lsm-tree]]
- [[sparse-index]]

---

```
OLTP workload                       OLAP workload
─────────────                       ─────────────
тысячи мелких запросов/сек          единицы гигантских scan'ов/сек
read + write вперемешку             почти только read
ms latency (user-facing)            секунды-минуты latency (отчёты)
ACID транзакции                     eventual ок

        │                                   │
        ▼                                   ▼

Row-oriented storage                Column-oriented storage
┌──────────────────────┐            ┌──────┐ ┌──────┐ ┌──────┐
│ row 1: a, b, c, d, e │            │col_a │ │col_b │ │col_c │
│ row 2: a, b, c, d, e │            │ a1   │ │ b1   │ │ c1   │
│ row 3: a, b, c, d, e │            │ a2   │ │ b2   │ │ c2   │
└──────────────────────┘            │ a3   │ │ b3   │ │ c3   │
                                    └──────┘ └──────┘ └──────┘
SELECT * WHERE id=2:                AVG(col_c):
  читаем 1 строку целиком             читаем ТОЛЬКО col_c

Postgres, MySQL, Cassandra          ClickHouse, Snowflake, BigQuery,
CockroachDB                         Redshift, DuckDB, Druid
```

**TL;DR:** OLTP — операционная БД для приложения (много мелких транзакций, ms-latency, ACID). OLAP — аналитическая БД для отчётов (мало гигантских scan'ов, секунды-latency). Workload диктует storage: OLTP обычно row-oriented, OLAP — column-oriented. Это два разных мира со своими СУБД, обычно соединяются через ETL/CDC.

## Что такое OLTP

**Online Transaction Processing** — операционная БД, обслуживающая приложение в реальном времени.

- Юзер кликнул → INSERT/UPDATE/SELECT.
- Запросов **много**: тысячи–миллионы/сек.
- Каждый запрос **мелкий**: 1–100 строк.
- Latency критична: ms-range, иначе UI лагает.
- Mix read + write.
- ACID транзакции обязательны.

Примеры: Postgres, MySQL, Cassandra (как операционное), CockroachDB, MongoDB.

## Что такое OLAP

**Online Analytical Processing** — аналитическая БД для отчётов, BI, ML.

- Аналитик запустил запрос → агрегат по миллиардам строк.
- Запросов **мало**: единицы–десятки/сек на кластер.
- Каждый запрос **гигантский**: сканирует терабайты.
- Latency секунды-минуты — норма.
- Почти только read.
- Транзакции обычно не нужны.

Примеры: ClickHouse, Snowflake, BigQuery, Redshift, DuckDB, Apache Druid.

## Сравнение

| Метрика | OLTP | OLAP |
|---|---|---|
| Запросов/сек | тысячи–миллионы | единицы–десятки |
| Размер запроса | мелкий (1–100 строк) | гигантский (млн–млрд строк) |
| Доля read | ~70% | ~99% |
| Latency target | ms | секунды–минуты |
| Concurrent users | тысячи | десятки |
| Транзакции | ACID критичны | eventual обычно ок |
| Размер БД | GB–TB | TB–PB |
| Layout данных | **row-oriented** | **column-oriented** |
| Индексы | много, точечный лукап | мало, scan |
| Compression | средняя | очень высокая |
| Кому | приложение | аналитики, BI, ML |

## Row vs Column — физическая разница

**Row-oriented**: строка лежит на диске целиком, подряд.

```
[id=1, email=alice@a.com, name=Alice, age=30, country=US]
[id=2, email=bob@b.com,   name=Bob,   age=25, country=DE]
[id=3, email=eve@e.com,   name=Eve,   age=40, country=US]
```

`SELECT * FROM users WHERE id = 2` — нативно: один read, вся строка Bob.

`SELECT AVG(age) FROM users` — катастрофично: читаем **все колонки** всех строк, отбрасываем 4/5 данных.

**Column-oriented**: каждая колонка хранится отдельным файлом.

```
id.col:      [1, 2, 3, ...]
email.col:   ['alice@a.com', 'bob@b.com', 'eve@e.com', ...]
name.col:    ['Alice', 'Bob', 'Eve', ...]
age.col:     [30, 25, 40, ...]
country.col: ['US', 'DE', 'US', ...]
```

`SELECT AVG(age) FROM users` — нативно: читаем **только age.col**. На 100 GB таблице с 10 колонками → 10 GB I/O вместо 100 GB.

`SELECT * FROM users WHERE id = 2` — дорого: нужно собрать строку из N колоночных файлов.

Бонусы column store:
- **Compression в разы лучше**: одна колонка → один тип данных → высокая повторяемость. country.col ужимается dictionary-encoding'ом в 10× и больше.
- **Vectorized execution**: SIMD-инструкции обрабатывают массив значений одной колонки. В 10–100× быстрее scalar.
- **Late materialization**: фильтрация по одной колонке → потом достать другие колонки только для подходящих строк.

Минус: insert/update дорогой — обновить одну строку = тронуть N файлов. Поэтому column store обычно append-only batch-loads.

## Архитектура: где OLTP, где OLAP

```
┌──────────┐  transactional   ┌──────────────┐
│   App    │ ───writes───►    │  Postgres    │
│ (user-   │                  │  (OLTP)      │
│  facing) │ ◄───reads─────   │  row-store   │
└──────────┘                  └──────┬───────┘
                                     │ ETL / CDC
                                     │ (Debezium, Airbyte,
                                     │  Fivetran, dbt)
                                     ▼
                              ┌──────────────┐    analytics
                              │ ClickHouse / │ ◄────────── BI tools
                              │ Snowflake    │             (Metabase,
                              │ (OLAP)       │              Tableau,
                              │ column-store │              Looker)
                              └──────────────┘
```

OLTP — primary store приложения. OLAP — **копия** данных, реструктурированная под аналитику.

Способы передачи:
- **ETL** (batch): ночные джобы перегоняют данные.
- **CDC** (Change Data Capture, streaming): подписка на WAL OLTP-БД (Debezium), события идут в Kafka, OLAP их потребляет. Лаг ~секунды.

## HTAP — попытка гибрида

**Hybrid Transactional/Analytical Processing** — одна БД, оба workload'а.

- **TiDB**: row store TiKV (OLTP) + column store TiFlash (OLAP), replication между ними.
- **SingleStore** (бывший MemSQL).
- **Snowflake Unistore**.

На практике HTAP редко вытесняет специализированные решения — две специализированные БД обычно эффективнее одной универсальной.

## Storage engine × Orientation — независимые оси

Важно: storage engine (LSM/B-tree) **независим** от orientation (row/column). Это третья ось.

| Layout | Engine | Где |
|---|---|---|
| Row + B-tree | классика OLTP | Postgres, MySQL InnoDB, SQLite |
| Row + LSM | distributed OLTP | CockroachDB, Cassandra |
| Column + immutable | data warehouse | Snowflake, BigQuery, Redshift, Parquet |
| Column + LSM-like merge | streaming OLAP | ClickHouse MergeTree, Apache Druid |

[[lsm-tree]] и B-tree описывают **как организован поиск**. Row/column — **как организован layout**. Это разные оси, можно комбинировать.

ClickHouse MergeTree — отдельный интересный кейс: column-oriented + LSM-подобный merge tree. Используется [[sparse-index]] (primary key на каждый 8192-й granule).

## Не только в БД

OLTP/OLAP — частный случай общего разделения **operational vs analytical** в обработке данных:

- **Real-time monitoring** (Prometheus, Datadog metrics) vs **historical analytics** (BigQuery, ClickHouse).
- **Event-driven hot path** (Kafka stream processing, edge functions) vs **batch processing** (Spark, Hadoop).
- **L1/L2 CPU cache** (fast, small, write-through) vs **DRAM/disk** (slow, large, batch updates).
- **CDN edge** (low-latency reads) vs **origin** (full data, slower).
- **Application database** (per-request) vs **data lake** (analytics, ML training).

**Общее везде**: когда workload'ы фундаментально различаются по latency / throughput / read-write mix, **специализированные системы превосходят универсальные**. Универсальное решение неизбежно выбирает компромисс, в котором оба workload'а страдают. Разделение «оперативный путь vs аналитический путь» — один из самых стабильных архитектурных швов в data-системах, повторяется от микропроцессоров до планетарного-масштаба data warehouse.
