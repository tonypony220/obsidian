---
title: Interface vs Discriminated Union
tags: [concept, pattern, type-system, tradeoff]
date: 2026-04-12
---

# Interface vs Discriminated Union

moc: [[front-moc]]
next: [[algebraic-data-types]] [[exhaustiveness-check]] [[chrome-mode-fsm]] [[flat-vs-orthogonal-state]]

---

```
  Interface (полиморфизм)          Discriminated Union (tagged)
  ┌────────────────────┐           ┌────────────────────────────┐
  │   <<interface>>    │           │  type NavAction =           │
  │      Action        │           │   │{ type:'SELECT';...}     │
  │  execute(): State  │           │   │{ type:'GO_BACK' }       │
  └────────┬───────────┘           │   │{ type:'DESELECT';...}   │
           │                       └──────────────┬─────────────┘
   ┌───────┴────────┐                             │
   ▼                ▼                    ┌────────▼────────┐
┌──────────┐  ┌──────────┐              │  switch(action.type)  │
│SelectAct.│  │GoBackAct.│              │  case 'SELECT': …     │
│ execute()│  │ execute()│              │  case 'GO_BACK': …    │
└──────────┘  └──────────┘              └───────────────────────┘
 + новый вариант   - новая операция      - новый вариант   + новая операция
```

Два фундаментально разных подхода к работе с вариантами типов. Выглядят похоже (оба описывают "один из нескольких вариантов"), но логика обработки — противоположная.

## Interface (полиморфизм)

**Данные скрыты, интерфейс общий.** Вызывающий код не знает, что внутри — каждый тип сам решает, что делать.

```ts
interface Action {
    execute(state: NavState): NavState;
}

class SelectAction implements Action {
    execute(state) { /* своя логика */ }
}
class GoBackAction implements Action {
    execute(state) { /* своя логика */ }
}
```

Добавить новый вариант — просто: новый класс, старый код не трогаем (Open-Closed Principle). Добавить новую операцию — больно: нужно менять каждый класс.

## Discriminated Union (tagged union)

**Данные открыты, логика в одном месте.** Поле-дискриминант (тег) позволяет компилятору сужать тип:

```ts
type NavAction =
    | { type: 'SELECT'; profile: Profile }
    | { type: 'DESELECT'; profileId: number }
    | { type: 'GO_BACK' }
    | { type: 'GO_FORWARD' };

switch (action.type) {
    case 'SELECT':
        action.profile    // ✅ TS знает что поле есть
        action.profileId  // ❌ compile error
        break;
}
```

Добавить новую операцию (ещё один `switch`) — просто. Добавить новый вариант — нужно обойти все `switch`-блоки (но компилятор поможет через [[exhaustiveness-check]]).

### Union vs Discriminated Union vs Enum в TypeScript

`|` в TypeScript — просто union (объединение). Не всякий union — discriminated:

- **Union** — `string | number`, без тега. Различаем через `typeof`
- **Discriminated union** — union объектов с общим полем-тегом (`kind`, `type`). TS по нему сужает тип в `switch`
- **Enum** — набор именованных констант без данных: `enum Direction { Up, Down }`

Discriminated union мощнее enum — каждый вариант это отдельная структура данных.

**В Rust** `enum` — это как раз discriminated union (варианты с данными), а не как enum в TS/Java.

## Expression Problem

Нельзя одновременно легко добавлять и варианты, и операции:

| | Новый вариант | Новая операция |
|---|---|---|
| **Interface** | Легко (новый класс) | Тяжело (менять все классы) |
| **Discriminated Union** | Тяжело (менять все switch) | Легко (новый switch) |

## Где какой подход идиоматичен

**Interface** — ООП-языки: Java, C#, Kotlin, Go (`interface{}`), Python (ABC/протоколы).

**Discriminated Union** — языки с pattern matching: Haskell, OCaml, F#, Rust (`enum` + `match`), Swift, TypeScript (`type A | B` + `switch`), Kotlin (`sealed class`).

## Когда что выбирать

**Discriminated Union**, когда: набор вариантов фиксирован, нужно много операций, данные вариантов различаются. Примеры: события, экшены reducer'а, AST-узлы, команды.

**Interface**, когда: варианты часто добавляются, операция одна но реализации разные, важна инкапсуляция. Примеры: стратегии, адаптеры, middleware, плагины.
