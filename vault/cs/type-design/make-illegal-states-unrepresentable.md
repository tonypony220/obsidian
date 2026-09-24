---
title: Make Illegal States Unrepresentable
tags: [concept, principle, type-system]
date: 2026-06-07
---

# Make Illegal States Unrepresentable

moc: [[type-design-moc]]
back: [[type-driven-design]]
next:
- [[algebraic-data-types]]
- [[exhaustiveness-check]]
- [[total-vs-partial-functions]]

---

```
boolean-flag tangle                  ADT
───────────────────                  ───
{ loading, data, error }             | loading
   2 × 2 × 2 = 8 комбинаций          | success(data)
   валидных: 3                       | error(e)
   illegal: 5 (не ловятся типом)        возможных: 3, ровно нужные
```

**TL;DR:** если состояние бизнес-логически невозможно, оно должно быть синтаксически непредставимо в типе. Boolean-flag tangle — главный анти-паттерн, который этот принцип лечит. Переписывание во variant tree через [[algebraic-data-types]] сводит число возможных значений к числу валидных.

## Принцип

Yaron Minsky (Jane Street, OCaml): *«Make illegal states unrepresentable»* (2011). Сильная формулировка: если можно написать значение типа `T`, это значение бизнес-валидно. Каждый шаг от этого идеала — потенциальный баг + место для runtime-проверки.

Якорь-вопрос при дизайне типа: **«сколько возможных значений у этого типа, и сколько из них бизнес-валидны?»**. Если расхождение — рефакторить тип, пока разрыв не закроется (или не сузится до неустранимого минимума).

## Анти-паттерн: boolean-flag tangle

Несколько независимых boolean-флагов, описывающих **одно** состояние. Комбинаций — `2^n`, валидных — горстка.

### Пример 1 — состояние загрузки

Плохо:
```ts
type FetchState = {
  loading: boolean;
  data: User | null;
  error: Error | null;
}
```
2³ = 8 комбинаций. Валидных 3: `{loading}`, `{data}`, `{error}`. Illegal — `{loading: true, data: X}`, `{data: X, error: E}`, и т.д. Каждое использование требует проверки «а это вообще валидное?». Где-то забудешь — баг.

Хорошо:
```ts
type FetchState =
  | { kind: 'loading' }
  | { kind: 'success'; data: User }
  | { kind: 'error'; error: Error };
```
Ровно 3 значения. `data` существует **только** в `success` — нельзя случайно прочитать `state.data` в loading. Компилятор знает, что в `error`-ветке `data` нет.

### Пример 2 — состояние заказа (FSM)

Плохо:
```ts
type Order = {
  isPaid: boolean;
  isShipped: boolean;
  isCancelled: boolean;
  paidAt?: Date;
  shippedAt?: Date;
  trackingNumber?: string;
  cancelReason?: string;
}
```
2³ × optional-комбинации = десятки состояний. Кто следит, что `isShipped=true ⇒ paidAt` есть? Что `isCancelled` и `isShipped` взаимно исключены? Никто. Инварианты живут в головах + тестах + код-ревью.

Хорошо — FSM прямо в типе:
```ts
type Order =
  | { status: 'pending' }
  | { status: 'paid';      paidAt: Date }
  | { status: 'shipped';   paidAt: Date; shippedAt: Date; tracking: string }
  | { status: 'cancelled'; cancelledAt: Date; reason: string };
```
`paidAt` без `paid` — невозможно. `tracking` без `shipped` — невозможно. Переходы FSM выражены типом, не runtime-проверками.

## Red flags

Сигналы, что тип нарушает принцип:
- Два+ независимых boolean-флага, описывающих одно состояние
- Optional поля, которые «должны быть только если другое поле = X»
- Комментарии вида `// устанавливается только когда status === 'paid'`
- Defensive проверки в коде: `if (state.data && !state.loading) ...`
- Тесты вида «не должно быть состояния, где...»

Каждое такое — кандидат на переписывание через [[algebraic-data-types]].

## Когда принцип НЕ применять

- **External data на границе** — пришедший JSON может быть в любом виде. Парсим во well-typed форму один раз ([[parse-dont-validate]]), дальше принцип работает.
- **Чистая косметика без инвариантов** — `{ verbose: boolean, quiet: boolean }` для CLI флагов: да, можно `--verbose --quiet`, но это пользовательская ошибка, а не баг типов. Принципом тут можно злоупотребить.
- **Когда стоимость моделирования > выгоды** — для совсем эпизодических полей с тривиальной семантикой. Но это редко: обычно тангл флагов отращивает новые комбинации со временем.

## Не только в типах

Тот же принцип:
- **Database schema** — `NOT NULL` + CHECK constraints вместо «приложение всегда пишет валидное»; FK вместо «id указывает на существующую запись по соглашению»
- **State machine (XState, Statecharts)** — переход, не описанный в FSM, не происходит; состояние, не описанное, не существует
- **Protocol design** — gRPC/protobuf required-поля и `oneof` вместо optional с дисциплиной
- **API forms** — radio-buttons вместо двух чекбоксов с правилом «оба не могут быть включены»
- **Rust borrow checker** — `&mut T` гарантирует уникальность владения; data race структурно невозможен

Общее везде: **запретить на уровне структуры** дешевле и надёжнее, чем **проверять на уровне процедуры**.
