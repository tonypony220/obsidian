---
title: TyDD без ADT
tags: [concept, tradeoff, type-system]
date: 2026-06-07
---

# Type-Driven Design без ADT

moc: [[type-design-moc]]
back: [[type-driven-design]]
next:
- [[algebraic-data-types]]
- [[exhaustiveness-check]]
- [[make-illegal-states-unrepresentable]]
- [[parse-dont-validate]]

---

```
с ADT (Rust, TS, Kotlin, Swift, F#)             без ADT (Go, старая Java, C)
──────────────────────────────────              ──────────────────────────
type Order = Paid | Shipped | Cancelled         type Order struct {
                                                  Status      string
switch o.kind {                                   PaidAt      *time.Time
  case 'paid': ...                                ShippedAt   *time.Time
  case 'shipped': ...                             Tracking    *string
  case 'cancelled': ...                           CancelledAt *time.Time
}  ✅ compiler требует все ветки                }  ← bool-flag tangle, инвариант в голове
```

**TL;DR:** принципы TyDD универсальны, ADT — самый дешёвый способ их выразить. Без ADT (Go, старая Java, C) принципы те же, цена дисциплины растёт: больше boilerplate, больше runtime-проверок, меньше compile-time гарантий. ADT не делают TyDD *возможным*, они делают его *дешёвым*.

## Спектр поддержки языков

| Язык | ADT | Exhaustiveness | Стоимость TyDD |
|------|-----|----------------|----------------|
| Haskell, OCaml, F#, Elm | нативно | встроена в pattern matching | минимальная |
| Rust, Swift, Kotlin (`sealed`), Scala | нативно | встроена | минимальная |
| TypeScript | discriminated unions | через `never`-probe ([[exhaustiveness-check]]) | низкая |
| Java 17+ (sealed + pattern matching), C# 9+ | да | частичная | средняя |
| Python + mypy strict | `Union` + `Literal` + `match` | только compile-time через mypy | средняя |
| **Go** | **нет** | **нет** | **высокая** |
| C, старая Java | union + tag вручную | нет | очень высокая |

## Что остаётся доступным в любом языке

- **[[make-illegal-states-unrepresentable]]** — частично, через классы с приватным конструктором: `new EmailAddress(s)` бросает на невалидном, наружу выдаётся только корректное значение
- **[[parse-dont-validate]]** — да, через wrapper-класс со smart constructor (статический `Email.parse(s)`)
- **[[total-vs-partial-functions]]** — концептуально да; возвращать `Result`-like тип вручную (в Go — `(value, err)`), избегать throw
- **[[branded-types]]** — эмулируется wrapper-классом (тяжелее, чем `string & __brand` в TS, но работает)

## Что теряется без ADT

- **Дешёвые sum types** — нельзя сказать «либо A либо B либо C» одним типом. Эмуляция через наследование/интерфейсы — много кода, не выражает «закрытый набор»
- **Compile-time exhaustiveness** — без него `default: panic` это **runtime-страховка**, не **compile-time-гарантия**. Добавил вариант — приходится grep'ать по кодовой базе
- **«Расширил union — компилятор красный везде»** — главное преимущество ADT; см. [[exhaustiveness-check]]. Без него — поиск по grep + надежда на тесты
- **Compositional типы**: `Result<Option<User>, FetchError>` — банально пишется в Rust/TS, болезненно эмулируется в Go

## Go-пример боли

```go
type Order struct {
    Status      string     // "pending"|"paid"|"shipped"|"cancelled" — string, не closed set
    PaidAt      *time.Time // только для paid+shipped
    ShippedAt   *time.Time // только для shipped
    Tracking    *string    // только для shipped
    CancelledAt *time.Time // только для cancelled
}
```

В Go нельзя выразить в типе:
- `Status` — closed enum (любая строка валидна синтаксически)
- `PaidAt` существует ⇔ `Status ∈ {paid, shipped}`
- `ShippedAt` и `CancelledAt` взаимоисключны

Только runtime-инварианты + дисциплина в конструкторах + тесты + код-ревью.

### Workaround: interface + типы-варианты

```go
type OrderState interface{ isOrderState() }

type Pending   struct{}
type Paid      struct{ PaidAt time.Time }
type Shipped   struct{ PaidAt, ShippedAt time.Time; Tracking string }
type Cancelled struct{ CancelledAt time.Time; Reason string }

func (Pending) isOrderState()   {}
func (Paid) isOrderState()      {}
func (Shipped) isOrderState()   {}
func (Cancelled) isOrderState() {}

// Использование
switch s := orderState.(type) {
case Pending:   ...
case Paid:      ...
case Shipped:   ...
case Cancelled: ...
default: panic("unreachable")  // runtime, не compile-time
}
```

Лучше boolean-flag tangle, но:
- `default: panic` — runtime-страховка. Добавил `Refunded` — компилятор молчит
- `OrderState interface{}` — открытый: кто угодно может реализовать `isOrderState()` извне (sealed-trick через unexported method частично закрывает)
- Много кода для каждого варианта

## Java/Kotlin-эквиваленты

- **Kotlin** `sealed class` / `sealed interface` — полноценный ADT, exhaustiveness в `when` встроена
- **Java 17+** `sealed interface` + `switch` с pattern matching — близко к ADT
- **Старая Java** — Visitor pattern (классика GoF) — «бедного человека ADT»; exhaustiveness достигается через abstract метод визитора, который требует реализовать все варианты

## Python

С mypy strict + `Union` + `Literal` + `match`:
```python
from typing import Literal, Union
from dataclasses import dataclass

@dataclass
class Loading: kind: Literal["loading"] = "loading"
@dataclass
class Success: data: User; kind: Literal["success"] = "success"
@dataclass
class Error:   error: Exception; kind: Literal["error"] = "error"

State = Union[Loading, Success, Error]

def render(s: State) -> str:
    match s:
        case Loading(): return "..."
        case Success(data=d): return d.name
        case Error(error=e): return str(e)
```
Exhaustiveness — только в mypy (`assert_never`). Runtime — нет гарантий. Зависит от строгой настройки type-checker'а в CI.

## Эвристика при выборе языка

Если домен **варианто-богатый** (FSM, workflow, AST, error hierarchies) и стоимость багов высокая → язык с ADT окупает себя за недели. Go подходит, когда домен **процедурный** и варианты тривиальны (CRUD, сеть, инфраструктура), а потеря на ADT — приемлемая цена за runtime/деплой-простоту.

## Не только в языках программирования

- **SQL без CHECK constraints** = «без ADT»: схема разрешает невалидные комбинации, инвариант живёт в приложении
- **JSON без схемы** vs **OpenAPI / protobuf** — без схемы каждый consumer валидирует сам; со схемой генератор кода даёт типизированные клиенты
- **REST без status code discipline** vs **typed error responses** — `200 {ok: false, error: ...}` тангл против `4xx + Problem Details` discriminated union
- **Конфиги без типизации** (raw YAML) vs **типизированные конфиги** (Cue, Dhall, Pkl) — без схемы баги обнаруживаются в проде

Общее везде: **отсутствие выразительного типа = инвариант мигрирует в runtime + дисциплину + тесты**. Стоимость не исчезает, только смещается.
