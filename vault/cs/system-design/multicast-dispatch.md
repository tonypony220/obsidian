---
title: Multicast Dispatch
tags: [pattern, state-management, fsm, architecture, frontend]
date: 2026-06-11
---

# Multicast Dispatch

moc: [[front-moc]]
back: [[chrome-mode-fsm]]
next:
- [[flat-vs-orthogonal-state]]
- [[fsm-global-transitions]]
- [[interface-vs-discriminated-union]]

---

```
       dispatch(event)
             │
   ┌─────────┼─────────┐
   ▼         ▼         ▼
reducer A reducer B reducer C
   │         │         │
state A   state B   state C       ← независимые scopes:
URL       session   page            URL-bound / session / page

каждый pattern-matches своё, остальное → return state.
SELECT_PROFILE доходит до всех трёх → A→profile, B→pop, C→closed.
Координация через факт доставки, не через shared state.
```

**TL;DR:** один `dispatch` веерно отправляется в N независимых reducer'ов; каждый pattern-matches свои события, остальное игнорит. Cross-axis инвариант между FSM с разными жизненными циклами выражается в transition'е, а не в `useEffect` или render-guard'е.

## Постановка

Несколько FSM, которые нельзя слить в один reducer — у них разные scope существования:
- chrome — URL-bound, deep-linkable, переживает session
- overlay-stack — session-scope, не сериализуется в URL
- page-local — живёт только пока смонтирована страница

Слить в один reducer = общий state-shape с optional-полями «эти живут только когда X» (boolean-flag tangle, антипаттерн из [[interface-vs-discriminated-union]]).

Держать раздельно через `useEffect`-связи = координация через side-effect, цикл «когда A→profile, B надо закрыть» уезжает в render-фазу — race condition'ы и render-guard'ы вместо инварианта в типах.

## Решение

Один `dispatch`, веерно рассылающий event во все reducer'ы:

```ts
const dispatch = useCallback((event: UiEvent) => {
  chromeDispatch(event)
  overlayDispatch(event)
  pageLocalDispatch(event)
}, [chromeDispatch, overlayDispatch, pageLocalDispatch])
```

Каждый reducer pattern-matches что ему интересно:

```ts
// page-local reducer
function reduce(state, event) {
  switch (event.type) {
    case 'OPEN_FILTERS':    return { kind: 'open' }
    case 'SELECT_PROFILE':  return { kind: 'closed' }   // координация с chrome
    default:                return state                 // no-op
  }
}
```

`SELECT_PROFILE` физически доходит до page-local reducer'а; невозможное состояние (`chrome=profile` ∧ `pageLocal=open`) становится **невыразимым в transition'ах**. Не render-guard, не useEffect — выражено машиной.

## Не `combineReducers` и не один switch

**`combineReducers` (Redux):** каждый reducer тоже видит каждое action — broadcast часть та же. Отличие — `combineReducers` слайсит **общий** state-объект, каждый reducer владеет своим слайсом. Здесь state-shapes полностью независимы, не slices одного.

**Один большой switch:** требует общего state-shape. Если у FSM разные жизненные циклы (URL vs session vs page) — общий state будет иметь optional-поля под каждый scope, что вырождается в флаги.

Multicast — между: независимый state (как N отдельных reducer'ов), общий event channel (как один большой switch).

## Когда применять

- ≥2 FSM с разными scope (URL-bound vs session vs page-local)
- Cross-axis инвариант («когда A→X, B должен закрыться»)
- Хочется выразить инвариант в transition reducer'а, не в `useEffect`/render-guard

Не применять, если:
- FSM реально одна — это один reducer, плоский или ортогональный ([[flat-vs-orthogonal-state]])
- FSM независимы и не координируются — отдельные `useReducer`'ы без shared dispatch достаточно

## Правило: page-local FSM, координирующая UI с chrome — третий reducer в общем dispatch

Изолированный `useReducer` в компоненте — только для состояния, **не пересекающегося** с chrome (локальный hover, focus, scroll position). Как только page-local FSM начинает слушать cross-axis события (`SELECT_PROFILE`, `HYDRATE_FROM_URL`, `USER_DISMISSED_PICKER`) — она подписывается на общий dispatch, а не живёт отдельным `useReducer` в обход.

Иначе появится третья невидимая FSM, и координация поедет в render-уровень mutex'ами и `useEffect`-связями.

## Концепция шире

«Один event → N независимых обработчиков, каждый pattern-matches своё»:

- **Event sourcing / CQRS** — один domain event → N read-model projections, каждая обновляется независимо.
- **Actor model** — message доходит до подписчиков, каждый actor сам решает реакцию; общего state нет.
- **Pub/Sub** — топик → N subscribers, никто не знает про остальных.
- **Elm Architecture composed update** — parent `Msg` через `Cmd.map` делегируется в child update'ы.

Общее везде: **N независимых state-машин, координация через факт доставки события, не через shared state**.
