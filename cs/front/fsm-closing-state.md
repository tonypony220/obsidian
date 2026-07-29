---
title: FSM closing-state — анимация exit'а как first-class состояние
tags: [pattern, fsm, state-management, animation, frontend]
date: 2026-04-23
---

# FSM closing-state

moc: [[front-moc]]
back: [[chrome-mode-fsm]]
next:
- [[on-exit-complete-callback]]
- [[presence-pattern]]
- [[fsm-global-transitions]]

---

```
 picking ──► confirming ──► validating ──► closing ──► idle
   ▲                                          │           │
   └───────── USER_DISMISSED_PICKER ───────────┘           │
                                                           │
   closing: данные ещё живы (draft, coords, filters),      │
            view рендерит exit-анимацию из них             │
                                                           │
   animationend → DISMISS_ANIMATION_DONE → ──────── idle ──┘
                                           данные очищены
```

**TL;DR:** если FSM атомарно переходит в `idle`, view теряет данные до того как проиграет exit-анимацию — получается пустой кадр или обрыв анимации. Решение: добавить в FSM отдельное состояние `closing`, в котором данные ещё живы. Exit играется от этих данных. Когда анимация завершилась — событие `DISMISS_ANIMATION_DONE` переводит FSM в `idle` и очищает данные. Это расширение паттерна [[on-exit-complete-callback]] на случай, когда источник данных exit'а — сам FSM, а не caller.

## Когда одного onExitComplete достаточно

[[on-exit-complete-callback]] решает sequential exit-then-next, если данные для exit-view живут **в caller'е**:

```tsx
<ProfileSelector profiles={profiles} onSelect={...}>
  ...
</ProfileSelector>
```

`profiles` хранит родитель, они никуда не деваются пока Sheet играет exit. Caller держит pending-намерение в ref'е (см. [[pending-action-ref-latch]]) и дождётся `onExitComplete`.

## Когда этого мало

Случай location-picker'а:

```ts
type ChromeMode =
  | { kind: 'idle' }
  | { kind: 'location-picker'; step, draft, coords, filters }
  | { kind: 'profile'; ... }
```

Данные для picker-view — `draft`, `coords`, `filters` — живут **в самом FSM**, в поле `chromeMode`. View рендерит их как:

```tsx
function renderChrome(mode) {
  switch (mode.kind) {
    case 'location-picker':
      return <LocationPicker draft={mode.draft} coords={mode.coords} />
    case 'idle':
      return null;
  }
}
```

Что ломается при атомарном `USER_DISMISSED_PICKER → idle`:

1. FSM: `kind: 'location-picker' → kind: 'idle'`.
2. React ре-рендерит: `renderChrome` возвращает `null`.
3. Picker-view **пропал из дерева в тот же кадр**. Exit-анимация не сыграла.
4. Даже если Presence обернёт — данных уже нет, `mode.draft` недоступен.

## Решение: closing как отдельное состояние

Расширяем FSM:

```ts
type ChromeMode =
  | { kind: 'idle' }
  | { kind: 'location-picker';
      step: 'picking' | 'confirming' | 'validating' | 'closing';
      draft, coords, filters }
```

Переходы:

```
USER_DISMISSED_PICKER     : любой-step → step: 'closing'
DISMISS_ANIMATION_DONE    : closing    → kind: 'idle' (данные очищены)
```

В `closing` поля `draft`/`coords`/`filters` **остаются**. View продолжает рендерить picker:

```tsx
case 'location-picker':
  const isClosing = mode.step === 'closing';
  return (
    <Presence present={!isClosing} onExitComplete={() => dispatch({ type: 'DISMISS_ANIMATION_DONE' })}>
      <LocationPicker draft={mode.draft} coords={mode.coords} />
    </Presence>
  );
```

Жизненный цикл:

```
t=0       user click X → dispatch USER_DISMISSED_PICKER
t≈0       FSM: step='closing', draft/coords жив
t≈0       Presence видит present=false → data-state="closed"
t≈0       exit-анимация стартует
t≈150ms   animationend → onExitComplete
t≈150ms   dispatch DISMISS_ANIMATION_DONE
t≈150ms   FSM: kind='idle' (draft/coords обнулены)
t≈150ms   renderChrome вернул null, DOM чистый
```

## Inversion of control на уровне FSM

Обычный mental model: **состояние** — это «где я сейчас», **переход** — атомарный скачок.

С `closing` добавляется: **состояние — это ещё и "где я задержался, пока кто-то снаружи скажет, что готово продолжать"**. FSM ждёт подтверждения от view-слоя.

Это та же инверсия, что в [[on-exit-complete-callback]], только поднятая уровнем выше — из компонента в FSM.

## Варианты FSM-расширения

**Отдельный флаг `isClosing: boolean`:**

```ts
{ kind: 'location-picker'; step; draft; isClosing: boolean }
```

Работает, но **флаг + step — параллельные SOT**. Возможно некорректное состояние `step: 'picking' && isClosing: true`. Лучше — единое поле `step` с вариантом `'closing'`.

**Отдельный top-level kind `closing-picker`:**

```ts
{ kind: 'closing-picker'; draft; coords }
```

Работает в простых случаях, но закрытие — это шаг жизненного цикла picker'а, а не другой режим. Держать внутри `kind: 'location-picker'` логичнее и даёт переиспользование при нескольких sub-closing'ах.

## Когда usage не нужен

- **Один overlay без фиксированных данных в FSM** → хватит [[on-exit-complete-callback]] + [[pending-action-ref-latch]].
- **Фоновая анимация** (тост, toast-очередь) — там exit не блокирует следующий шаг, driver очереди живёт сам по себе.
- **Если exit мгновенный** (`prefers-reduced-motion` или нет анимации) — FSM может прыгать напрямую, `closing` вырождается в пустой переход.

## Связь с chrome-mode-fsm

Это прямое расширение паттерна [[chrome-mode-fsm]]:
- Раньше `step` описывал семантические шаги взаимодействия (`picking`, `confirming`).
- Теперь `step` описывает **и** фазу жизненного цикла (`closing`) — exit-анимация становится first-class гражданином FSM, а не побочным эффектом view.

## Концепция шире

«Промежуточное состояние для дренажа» встречается везде, где есть жизненный цикл с асинхронным завершением:

- **Kubernetes `Terminating`** — pod в статусе `Terminating` пока отрабатывают preStop-хуки и graceful shutdown, только потом удаляется из apiserver.
- **TCP `TIME_WAIT`/`CLOSE_WAIT`** — соединение в промежуточном состоянии пока не обработаны in-flight-пакеты.
- **Database `draining`** — пул соединений в dra ining-фазе не берёт новые, но ждёт завершения открытых транзакций.
- **Service mesh circuit breaker `half-open`** — не полностью open и не closed, промежуточная проба.
- **Git `rebase in progress`** — репо в промежуточном состоянии, часть операций запрещена до `--continue` или `--abort`.

Общее везде: **корректное завершение — не атомарный акт, а отдельная фаза с собственными инвариантами**. Моделировать её явным состоянием честнее, чем прятать в флаги или side-effect'ы.
