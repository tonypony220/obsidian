---
title: Stack vs Step Navigation
tags: [pattern, navigation, fsm, tradeoff, cheatsheet]
date: 2026-04-11
---

# Stack vs Step Navigation

moc: [[front-moc]]
next: [[chrome-mode-fsm]] [[in-memory-navigation-stack]]

---

```
  Step (линейный путь)              Stack (ветвящийся путь)
  ┌─────────┐  NEXT  ┌──────────┐   preview(A) → connections(A)
  │ picking │──────► │confirming│            ↓
  └─────────┘        └──────────┘   preview(B) → connections(B)
       ▲                  │                  ↓
       └──── BACK_IN ─────┘         chat(B↔C)
                                    ┌─────┐ pop  ┌──────────────┐
  step: 'picking'|'confirming'      │chat │ ───► │connections(B)│
  (предыдущий всегда известен)      ├─────┤      └──────────────┘
                                    │conn │   BACK = stack.slice(1)
                                    ├─────┤
                                    │prev │
                                    └─────┘
```

Внутри одного состояния FSM ([[chrome-mode-fsm]]) часто нужно хранить "историю" — откуда пришёл, куда вернуть по "назад". Есть три способа, и выбор зависит от того, **ветвится ли** путь к текущему экрану.

## Правило большого пальца

| Свойство операции | Что использовать |
|---|---|
| Линейная: A → B → C, "назад" всегда в предыдущий шаг | `step: 'A' \| 'B' \| 'C'` — поле внутри `kind` |
| Ветвящаяся, фиксированная глубина | Несколько полей: `step`, `history: Step[]` ограниченного размера |
| Произвольная глубина, неопределённые пути | Стек `ProfileStackView[]` внутри `kind` |
| Принципиально разные режимы | **Разные `kind`** — это не стек, это альтернативы |

## Линейный случай → step достаточно

```ts
{
  kind: 'location-picker'
  step: 'picking' | 'confirming'
  draft: Profile
  firstLocation?: Coord
}
```

Редьюсер:
```ts
case 'location-picker': {
  if (event.type === 'BACK_IN_LOCATION_PICKER' && state.step === 'confirming')
    return { ...state, step: 'picking', confirmState: null }
  if (event.type === 'PICK_AT_CENTER' && state.step === 'picking')
    return { ...state, step: 'confirming', confirmState: { coord: event.coord } }
  if (event.type === 'CANCEL_LOCATION_PICK')
    return { kind: 'idle' }
}
```

Никакой истории не нужно: из `confirming` "назад" **всегда** ведёт в `picking`. Детерминированный обратный путь = оверкилл для стека.

## Когда нужен стек

Стек оправдан, когда "предыдущая точка" зависит от того, **как** ты попал в текущую, а не только от того, где сейчас находишься.

Пример — навигация по профилям:
```
preview(A) → connections(A) → preview(B) → connections(B) → chat(B ↔ C)
```

Из `chat(B ↔ C)` "назад" может вести в `connections(B)`, а могло бы и в `full(B)`, если бы пришли другим путём. Без истории не угадаешь, поэтому:
```ts
{
  kind: 'profile'
  stack: [ProfileStackView, ...ProfileStackView[]]
}
```

`BACK` тогда — это `stack.slice(1)`, и если стек опустел — переход в `idle` (или обратно в `filter-results` для `filter-results-preview`).

## Гибрид: stack + step в одном kind

Нормально иметь оба поля, когда они описывают разные оси:

```ts
{
  kind: 'deploy'
  stack: [ProfileStackView, ...]        // откуда пришёл, куда вернуться при полном закрытии
  step: 'name' | 'photo' | 'link' | 'main'  // внутренний шаг формы
}
```

Два разных `BACK`:
- `BACK_IN_DEPLOY_FORM` — шаг в форме (`step` → предыдущий)
- `CANCEL_DEPLOY` — выход из `kind`, возврат по `stack`

## Две семантики "назад" — НЕ путать

Главная ошибка — запихнуть обе в одно событие `BACK`:

1. **Шаг назад внутри операции** — "я нажал Set location, но точка не та, хочу поправить". Возврат `confirming → picking`, модальный пикер не закрывается, `mapMode` не сбрасывается.
2. **Отмена всей операции** — закрыть пикер и вернуться откуда пришёл (`idle`, `filter-results`, ...).

Должны быть **два разных события**:
```ts
| { type: 'BACK_IN_LOCATION_PICKER' }   // (1) step назад внутри kind
| { type: 'CANCEL_LOCATION_PICK' }       // (2) выйти из kind полностью
```

Смешение = баги вроде "Escape в подтверждении стёр всю сессию". UI-уровень (Back-кнопка vs крестик vs hardware back) мапится в разные events, редьюсер их обрабатывает по-разному.

## Инвариант через типы

Если `step: 'confirming'` обязательно означает наличие `confirmState`, это можно выразить дискриминирующим union'ом внутри `kind`:

```ts
type LocationPickerMode =
  | { kind: 'location-picker'; step: 'picking';    draft: Profile; confirmState?: never }
  | { kind: 'location-picker'; step: 'confirming'; draft: Profile; confirmState: LocationPickerConfirmState }
```

Тогда доступ к `confirmState` типобезопасен только в правильном `step`.
