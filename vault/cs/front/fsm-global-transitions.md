---
title: FSM global transitions — переходы из любого kind
tags: [pattern, fsm, state-management, frontend]
date: 2026-05-07
---

# FSM global transitions

moc: [[front-moc]]
back: [[chrome-mode-fsm]]
next:
- [[fsm-closing-state]]
- [[interface-vs-discriminated-union]]
- [[multicast-dispatch]]

---

```
                ┌──────────────┐
   any kind ──► │   reducer    │
                │              │
                │  ┌────────┐  │  ── global handler ──
                │  │ if ESC │──┼──► { kind: 'idle' }      ◄── срабатывает
                │  └────────┘  │                              из ЛЮБОГО kind
                │      │       │
                │      ▼       │
                │  switch(kind)│  ── per-kind transitions ──
                │   ┌── A ──┐  │
                │   ├── B ──┤  │
                │   └── C ──┘  │
                └──────────────┘
```

**TL;DR:** события, которые должны вести себя одинаково из любого `state.kind` (cancel, ESC, fatal error, sign-out), обрабатывать **одним глобальным условием в начале редьюсера** — до `switch (state.kind)`. Это catch-all правило, которое поверх per-kind переходов. Без него ветка cancel дублируется в каждом `case` и легко забывается в новом kind.

## Проблема: cross-kind дублирование

Без глобального handler'а:

```ts
function reduce(state: ChromeMode, event: UiEvent): ChromeMode {
  switch (state.kind) {
    case 'location-picker':
      if (event.type === 'USER_PRESSED_ESC') return { kind: 'idle' };
      // ...
    case 'profile':
      if (event.type === 'USER_PRESSED_ESC') return { kind: 'idle' };
      // ...
    case 'editing-filter':
      if (event.type === 'USER_PRESSED_ESC') return { kind: 'idle' };
      // ...  ← забыл в новом kind → ESC не работает
  }
}
```

Инвариант «ESC всегда закрывает» нигде не записан, размазан по N веткам. Регрессия — вопрос времени.

## Решение 1: global handler перед switch

```ts
function reduce(state: ChromeMode, event: UiEvent): ChromeMode {
  // --- Global transitions (работают из любого kind) ---
  if (event.type === 'USER_PRESSED_ESC') return { kind: 'idle' };

  // --- Per-kind transitions ---
  switch (state.kind) {
    case 'location-picker': return reducePicker(state, event);
    case 'profile':         return reduceProfile(state, event);
    // ...
  }
}
```

Одно место — нельзя забыть. Инвариант выражен кодом, не дисциплиной.

**Именование событий — event-style, не command-style.** `USER_PRESSED_ESC` (что случилось) лучше, чем `GO_TO_IDLE` (команда). Что это означает на FSM-уровне — решает сам редьюсер. Один и тот же ESC из location-picker может вести в `idle`, а из profile — в filter-results (parent kind), и редьюсер вправе это интерпретировать по-разному.

## Решение 2: per-kind с fallback (bubbling)

Если для некоторых kind'ов cancel должен делать особое (показать «discard?», сохранить draft) — даём kind'у шанс перехватить, потом падаем в global:

```ts
function reduce(state: ChromeMode, event: UiEvent): ChromeMode {
  const handled = perKindReducer(state, event);
  if (handled !== state) return handled;        // kind перехватил

  if (event.type === 'USER_PRESSED_ESC')        // global fallback
    return { kind: 'idle' };
  return state;
}
```

Это child → parent event bubbling из statechart-моделей (XState). Child может «поглотить» событие.

## Когда parent ≠ idle: map переходов

Если разным kind'ам нужны разные «куда возвращаться» при cancel — это всё ещё одно место, просто с map'ом:

```ts
const PARENT_ON_CANCEL: Record<ChromeMode['kind'], ChromeMode['kind']> = {
  'location-picker': 'idle',
  'profile-picker':  'filter-results',
  'editing-filter':  'filter-results',
  // ...
};
if (event.type === 'USER_PRESSED_ESC') {
  return { kind: PARENT_ON_CANCEL[state.kind] };
}
```

Иерархия выражается данными, а не разветвлением кода.

## Если cancel запускает анимацию exit'а

Глобальный переход не обязан вести напрямую в `idle`. Если данные нужны view'у для exit-анимации — global handler ведёт в [[fsm-closing-state]]:

```ts
if (event.type === 'USER_PRESSED_ESC' && state.kind !== 'idle') {
  return { ...state, step: 'closing' };   // данные сохраняются
}
```

Затем `DISMISS_ANIMATION_DONE` (от view) переводит в `idle`. Cross-kind cancel и closing-фаза — ортогональны: один говорит «откуда срабатывает», второй — «как корректно завершить».

## Side-effects при выходе — не в редьюсер

Редьюсер — чистая функция. Если на выходе из kind нужно abort fetch / close SSE / снять подписку — это в эффект-слое поверх FSM, реагирующем на смену `state.kind`:

```ts
useEffect(() => {
  if (prevKind === 'location-picker' && state.kind === 'idle') {
    abortGeocodingRequest();
  }
}, [state.kind]);
```

Mixing side-effects с редьюсером ломает воспроизводимость и тесты.

## Концепция шире

«Catch-all правило поверх специфичных переходов» — паттерн, не специфика FSM:

- **Middleware fall-through** (Express/Koa, Rack, ASP.NET) — error handler в конце цепочки ловит всё, что не обработано.
- **Exception handlers** — `catch (Throwable)` в самом верху ловит то, что не поймали более узкие `catch`.
- **Routing** — wildcard-роут `*` / `404` после конкретных путей.
- **Pattern matching** — `default` / `_` ветка после специфичных pattern'ов (`switch`, Rust `match`, Haskell, Erlang).
- **OS signal handlers** — `SIGTERM` обрабатывается процессом одинаково независимо от текущей фазы работы.
- **Firewall / ACL** — `deny all` в конце цепочки правил, после allow-исключений.

Общее везде: **catch-all правило поверх специфичных**. Порядок важен (специфичное первым, общее последним), и центральное расположение catch-all делает инвариант видимым, а не размазанным.
