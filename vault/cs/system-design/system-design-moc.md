---
title: System Design MOC
tags: [moc]
date: 2026-06-21
---

# System Design — MOC

next:
- [[architecture-stack]]
- [[test-strategy-moc]]
- [[enforcement-moc]]

---

System Design = принятие решений о структуре системы под конкретные требования (functional + non-functional + constraints). Архитектура — следствие, не отправная точка. Ветка наращивается; серая ссылка = ещё не разобрано.

## Точка входа

- [[architecture-stack]] — мета: где какой паттерн живёт, как слои стека сочетаются, fix left-to-right

## Структура кода (как режем проект)

- [[module-organization-patterns]] — Layered / Vertical Slice / Modular Monolith — как нарезать саму кодовую базу
- [[hex-architecture]] — Ports & Adapters: домен в центре, IO на границе
- [[driving-vs-driven-ports]] — интерфейс нужен только против runtime-стрелки (driven-сторона); driving-порт = сигнатура функции
- [[clean-architecture]] — Hex + явный слой Use Cases (entities vs application scenarios)
- [[functional-core-imperative-shell]] — дисциплина: домен = чистые функции, IO снаружи
- [[use-cases-as-named-operations]] — сценарии как отдельные классы, не ветки `if kind===`

## Границы, швы, инварианты

- [[seam]] — точка подмены поведения без правки caller'а; тактика, не архитектура
- [[middleware-patterns]] — конвейерная обработка запроса
- [[pin-by-construction]] — обязательность через тип результата, не через ручной guard
- [[multicast-dispatch]] — один dispatch → N reducer'ов; cross-axis инвариант в transition'е
- [[fitness-function]] — структурный инвариант репо как тест

## Deployment / runtime

- [[edge-vs-serverless]] — когда edge реально полезен
- [[cold-start]] — задержка первого запроса в serverless/контейнерах
- [[control-plane-data-plane]] — управление vs трафик; разные SLA, разные failure modes
- [[fail-open-vs-fail-closed]] — поведение при отказе зависимости

## API

- [[graphql]] — как работает и зачем нужен

## Traffic control

- [[sliding-window-rate-check]] — «≤ K в любом окне W» за O(1)/событие: сравнение с таймстемпом K назад
- [[rate-limiter-algorithms]] — семейство: log/ring/buckets (точные) vs fixed window/token bucket (O(1), приближение)
- [[counters-vs-events]] — почему Prometheus-counter не ловит burst: агрегация теряет информацию

## Storage layer

- [[lsm-tree]] — append-only движок хранения: memtable → SSTable → compaction; ортогонален модели данных (SQL/NoSQL); полный write/read path с примерами
- [[lsm-vs-btree]] — механика обеих структур, write/read path, write/read/space amplification
- [[bloom-filter]] — probabilistic membership test: «точно нет» vs «возможно есть», 10 бит/ключ → 1% FP
- [[sparse-index]] — индекс по блокам, не по элементам; иерархия «грубый в RAM + точный с диска»
- [[oltp-vs-olap]] — операционные транзакции vs аналитика; row vs column; storage engine × orientation как ортогональные оси
- [[fsync-and-durability]] — почему durable writes стоят дорого; page cache, group commit, цена на разных дисках
- [[wal-write-ahead-log]] — append-only лог намерений до изменения данных; recovery, checkpoint, основа ACID-commit'а

## Scalability (bottleneck axes) — раздел растёт

- [[write-throughput-vs-connection-count]] — две независимые оси нагрузки; gateway-слой и storage-слой скейлятся раздельно
- [[connection-state-cost]] — что жрёт одно WebSocket-соединение (FD, kernel buffers, user state)
- [[fan-out-on-write-vs-read]] — модель доставки для групп/каналов; точка перелома

## Media / streaming

- [[video-codec]] — intra/inter compression: почему 4K-стрим ~1000× меньше raw; кодек vs контейнер vs протокол

## Соседние MOC

- [[test-strategy-moc]] — как верифицировать дизайн (Walking Skeleton, coverage matrix)
- [[enforcement-moc]] — как удержать инвариант после изменения (pin, fitness function)
- [[engineering-process-moc]] — процесс работы над изменениями
