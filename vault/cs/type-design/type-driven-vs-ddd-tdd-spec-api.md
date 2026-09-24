---
title: TyDD vs DDD / TDD / Spec-Driven / API Design
tags: [concept, meta, type-system, methodology]
date: 2026-06-07
---

# TyDD ↔ DDD, TDD, Spec-Driven Dev, API Contract Design

moc: [[type-design-moc]]
back: [[type-driven-design]]
next:
- [[make-illegal-states-unrepresentable]]
- [[parse-dont-validate]]
- [[branded-types]]

---

```
                          корректность по построению
                                     │
                ┌────────────┬───────┴────────┬──────────────┐
                ▼            ▼                ▼              ▼
              TyDD          DDD              TDD          Spec-Driven
            (формальное   (моделирование    (поведение    (контракт ↔ реализация
             доказательство  домена)         через тесты)  как первоисточник)
             типом)
```

**TL;DR:** TyDD — техническая дисциплина (компилятор enforces); DDD — методология моделирования домена (включает ubiquitous language, контексты, агрегаты); TDD — техника проверки поведения через тесты; Spec-Driven — спека-первоисточник. Пересечения сильные: TyDD = тех-механизм для DDD; TyDD сокращает поверхность TDD; TyDD = формальное подмножество spec; PDV = boundary validation в API.

## DDD ↔ TyDD: техника реализации

**DDD** (Eric Evans, 2003) — моделирование домена в сложных бизнес-системах:
- Ubiquitous Language — единый словарь команды и кода
- Bounded Context — граница, внутри которой термины имеют одно значение
- Aggregate — кластер объектов с общим инвариантом
- Value Object — иммутабельная сущность без идентичности (`Money`, `Email`)
- Entity — объект с идентичностью (`User`, `Order`)
- Domain Event — факт, произошедший в домене

**Где совпадает:**

| DDD-понятие | TyDD-аналог |
|-------------|-------------|
| Value Object | [[branded-types]] + smart constructor + [[parse-dont-validate]] |
| Aggregate invariants | [[make-illegal-states-unrepresentable]] — illegal aggregate state непредставим |
| Bounded Context boundary | граница парсинга: внешний язык → внутренний типизированный |
| State of Entity (FSM) | sum type с дискриминатором; см. Order пример |
| Domain Event | discriminated union событий; exhaustiveness в обработчиках |
| Anti-Corruption Layer | парсер на границе bounded context'а |

**Где расходится:**
- DDD шире TyDD: включает стратегические паттерны (context maps, shared kernel), организационные аспекты (команды per bounded context), методологию discovery (event storming)
- TyDD технически глубже: формальные гарантии компилятора, exhaustiveness, total functions
- DDD без типов = дисциплина соглашений; TyDD = enforces дисциплину компилятором

**Связь:** DDD говорит **что** моделировать (домен, границы, инварианты); TyDD говорит **как** это выразить так, чтобы компилятор это защищал. DDD-практикам в OCaml/Rust/TS легче, чем в Go/PHP.

## TDD ↔ TyDD: комплементарные

**TDD** (Kent Beck): red → green → refactor. Тесты как driver дизайна.

| Ось | TDD | TyDD |
|-----|-----|------|
| Что проверяет | поведение | структуру, инварианты |
| Где проверка | runtime, на сэмплах | compile-time, exhaustive по типам |
| Сила проверки | конкретные входы | весь типовой домен |
| Слабость | не покрывает unsampled cases | не выражает behaviour (только shape) |
| Когда срабатывает | при запуске тестов | при компиляции |

**Не конкуренты:**
- Типы проверяют **всё, что закодировано в типах** — но только это
- Тесты проверяют **произвольное поведение** — но только на запущенных кейсах

**Как сочетаются:**
- TyDD **сокращает поверхность тестов**. Если хэндлер принимает `Email`, не нужен тест «что если email malformed?» — этот случай непредставим
- TyDD убирает тесты-инварианты (`должно быть paid && !cancelled`) — они тривиально доказаны типом
- Остаются тесты **бизнес-логики**: «при оплате создаётся payment record», «при отмене возвращаются средства»

**Анти-паттерн:** писать тест на то, что уже гарантировано типом (`expect(typeof res).toBe('number')` при сигнатуре `(): number`).

**Walking Skeleton** (см. test-strategy-general skill) и TyDD дружат: типы фиксируют контракты пайплайна, e2e/субкутан тесты проверяют что пайплайн прошёл туда-обратно.

## Spec-Driven Dev ↔ TyDD: типы — машинно-проверяемое подмножество спеки

**Spec-Driven** (см. project CLAUDE.md): сначала спецификация, потом код. Документация — первоисточник.

| Что выражает спека | Куда уходит в TyDD |
|---------------------|----------------------|
| Структура данных (формы, поля) | в типы напрямую |
| Закрытые наборы значений | sum types ([[algebraic-data-types]]) |
| Состояния и переходы | FSM как discriminated union |
| Формат входа (regex, шаблоны) | парсеры → [[branded-types]] |
| Инварианты («X не может без Y») | [[make-illegal-states-unrepresentable]] |
| Rationale, trade-offs, мотивация | **остаётся в ADR/spec** — типы это не выражают |
| Не-функциональные требования (latency, retries) | **остаётся в spec** |
| Поведение во времени (асинхронность, race conditions) | частично типы, в основном тесты + spec |

**Принцип локальности из CLAUDE.md:** контракты → типы, rationale → ADR, кросс-модульная архитектура → docs. TyDD реализует «контракты → типы». Spec **дополняет**, а не заменяется.

**Discriminated union = FSM-спека прямо в коде.** Бизнес-аналитик и компилятор смотрят на один и тот же текст. Это лечит главный риск spec-driven: «спека есть, но устарела» — спека-в-типах устаревать не может, иначе билд красный.

## API Contract Design ↔ TyDD: граница как место парсинга

См. skill `api-contract-design`. Ключевые места пересечения:

**Parse Don't Validate на HTTP-границе:**
- Request body → парсится один раз в route handler в типизированный domain object
- Path/query params → парсятся в `UserId`, `Pagination`, `IsoDate`
- Невалидный input → 400 + RFC 7807 Problem Details (типизированная ошибка)

**RFC 7807 Problem Details = типизированные ошибки:**
```ts
type Problem =
  | { type: 'https://errors.app/not-found'; title: string; status: 404 }
  | { type: 'https://errors.app/validation'; title: string; status: 400; errors: ValidationError[] }
  | { type: 'https://errors.app/rate-limit'; title: string; status: 429; retryAfter: number };
```
Поле `type` — дискриминатор. Клиент делает exhaustive switch по нему. Это discriminated union на уровне протокола.

**Branded types для протокольных значений:**
- `IdempotencyKey` — branded string, не путается с другими headers
- `Cursor` — opaque branded для pagination (клиент не парсит)
- `RequestId` — для correlation
- `JwtToken` — отличается от сырой строки

**API-контракт между двумя сервисами = parser-спека:** «вот форма входа, вот форма выхода, вот ошибки». OpenAPI/protobuf/GraphQL — машинно-читаемые spec, из которых генерируются типы. Это spec-driven + TyDD одновременно.

**Status code discipline** (см. api-contract-design):
- `2xx` ⇒ Body соответствует success schema (sum type ветка «успех»)
- `4xx/5xx` ⇒ Body соответствует Problem schema (sum type ветка «ошибка»)
- Один union на endpoint, не «иногда возвращает X, иногда Y без status code различия»

**Async long-running (202 + poll):** state of operation = discriminated union (`pending | running | succeeded | failed`).

## Сводная таблица

| Дисциплина | Что делает | Где живёт | Кто enforces | Связь с TyDD |
|------------|------------|-----------|---------------|--------------|
| **TyDD** | Корректность по построению через типы | в типах/коде | компилятор | сама |
| **DDD** | Моделирование домена | модель + код | команда + соглашения | TyDD = тех-механизм для DDD |
| **TDD** | Корректность поведения через тесты | в тестах | CI на каждом запуске | TyDD сокращает поверхность тестов |
| **Spec-Driven** | Спека — первоисточник | в spec + ADR | дисциплина + дрейф-чеки | типы = формальное подмножество спеки |
| **API Contract** | Контракт сервисов | в схеме (OpenAPI/protobuf/...) | gateway + кодген + tests | PDV на границе = ядро contract design |

## Эвристика выбора

- Нужны **формальные гарантии для invariants** → TyDD первичен
- Нужно **общее понимание домена в команде** → DDD первичен; TyDD реализует
- Нужна **уверенность в поведении при изменениях** → TDD первичен; TyDD убирает часть тестов
- Нужна **документация-первоисточник для не-разработчиков** → Spec-Driven первичен; типы поглощают то, что машинно-выразимо
- Нужна **граница между сервисами** → API Contract; PDV + branded types обязательны на ней

Не выбирать одно — комбинировать. Каждая дисциплина закрывает свою плоскость; пересечения усиливают, не дублируют.
