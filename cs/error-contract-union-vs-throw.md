---
title: Контракт ошибки — union vs throw
tags: [concept, pattern, type-system, error-handling]
date: 2026-05-07
---

# Контракт ошибки — union vs throw

moc: [[fp-patterns-moc]]
back: [[result-either]]
next:
- [[interface-vs-discriminated-union]]
- [[exhaustiveness-check]]
- [[algebraic-data-types]]

---

```
union (ошибка-как-значение)        throw (ошибка-как-канал)
─────────────────────────          ────────────────────────
   Promise<T | ErrorResponse>        Promise<T>
        │                                │
        ↓                          resolve / reject
   caller: if (isError) ...              │
        │                                ↓
   TS сужает тип                framework ловит rejection
        │                                │
        ↓                                ↓
   ветви типизированы           state machine: { error }
```

**TL;DR:** union кодирует ошибку в типе, throw — в control flow. Императивный caller (рядом с вызовом) предпочитает union: типы сужаются, забыть нельзя, код линейный. Декларативный caller (framework владеет потоком) требует throw: однородный сигнал rejection интегрируется в его state machine.

## Два контракта

**throw** — ошибка как событие control flow:
- Сигнатура: `Promise<T>`. Какие ошибки бросаются — **не часть типа**.
- Caller: `try/catch` или unwind до ближайшего обработчика выше по стэку.
- TS: `catch (err: unknown)` — сужения нет, нужен `instanceof` + cast.

**union** — ошибка как значение:
- Сигнатура: `Promise<T | ErrorResponse>`. Все возможные ошибки — **в типе**.
- Caller: `if (isError(r)) … else …`. Promise всегда resolves.
- TS: после type guard оба варианта сужены автоматически.

См. [[result-either]] — это тот же паттерн в систем-типах: ADT `Result<T, E>`.

## Кто владеет control flow после ошибки

Главный вопрос. Ответ определяет, какой контракт уместен.

### Декларативный caller — throw

```
queryFn(): Promise<T>
      │
      ↓ throw
React Query state machine
      │
      ↓
{ status: 'error', error }
```

Framework композирует операции и прокидывает ошибку мимо них в отдельный канал. Ему нужен **однородный сигнал** «провалилось», независимый от прикладных типов. `Promise.reject` — такой сигнал на уровне платформы.

Union тут технически не работает: `{ error: '...' }` и `{ id: 123 }` на уровне Promise неразличимы — оба resolved. Framework не знает прикладных типов и не угадает «вот это значит провал». Поэтому контракт `queryFn`: **throw для ошибки, resolve для успеха** — это протокол producer↔framework.

### Императивный caller — union

```
onClick:
  1. await api(...)
  2. if error → toast + reset
  3. if ok → update state
```

Между API-вызовом и обработкой — 0 кадров фреймворка. Unwind'ить нечего: `catch` стоит в той же функции, что `try`.

С throw:
```ts
try {
  const r = await submitFeedback(...);
  setFeedback(r); setLoading(false);
} catch (err) {
  setError((err as ApiError).message); setLoading(false);
}
```
- `err: unknown` — TS не сужает. ApiError, NetworkError, AbortError, DOMException — в одну кучу.
- Сигнатура `Promise<T>` не говорит, **какие** ошибки возможны. В TS нет `throws`.
- Забыть `try/catch` = silent unhandledrejection, компилятор не напомнит.
- Cleanup дублируется в обеих ветках или выносится в `finally`.

С union:
```ts
const r = await submitFeedback(...);
if (isFeedbackError(r)) { setError(r.error); return; }
setFeedback(r);
```
- TS сужает `r` после guard'а — обе ветки типизированы.
- `r.id` без guard'а = ошибка компиляции. Забыть нельзя.
- Тип возврата документирует **какие именно** ошибки бывают.
- Линейный flow без исключений.

## Инвариант

|                              | union                           | throw                           |
|------------------------------|---------------------------------|---------------------------------|
| Где живёт ошибка             | в типе значения                 | в control flow                  |
| Кто принуждает обработку     | компилятор (тип)                | runtime (unhandled rejection)   |
| Сужение в TS                 | да, через guard                 | нет (`err: unknown`)            |
| Декларативный pipeline       | не композится                   | композится через канал rejection|
| Дистанция caller→error       | 0–1 кадр                        | произвольная (unwind стэка)     |

## Эвристика выбора

- **throw**: когда **не ты** обрабатываешь ошибку — её ловит framework, error boundary, middleware. Один escape unwind'ит много кадров.
- **union**: когда **ты сам** обрабатываешь ошибку рядом с вызовом. Унифицированная обработка не нужна; нужна типизация и явный branching.

Если между вызовом и обработкой стоит **framework** — throw. Если стоит **только пользовательский код** — union.

## Не только в JS/TS

Тот же дуализм встречается везде:

- **Go**: `(value, err)` — union; `panic` — throw для непредвиденного. Императивный caller проверяет `err != nil`.
- **Rust**: `Result<T, E>` — union; `panic!` — throw для bug'ов. Оператор `?` — сахар над union для unwind по чейну.
- **Swift**: `throws`-функции с `try` на каждом call-site — гибрид (throw, но компилятор требует пометить точку unwind).
- **HTTP**: status codes в ответе (union) vs network error (throw на уровне TCP/TLS). 4xx/5xx — успешно полученный ответ-с-ошибкой; ECONNREFUSED — нет ответа.
- **gRPC**: status code в трейлерах — union; transport error — throw.
- **Erlang**: `{ok, V} | {error, R}` — union; `exit/throw` — для supervision tree (framework ловит и рестартит процесс).

Общее везде: **ошибка как значение** оптимизирует точечную локальную обработку и типизацию; **ошибка как событие** оптимизирует unwind через много кадров и интеграцию с framework'ом, который routes ошибки в отдельный канал.
