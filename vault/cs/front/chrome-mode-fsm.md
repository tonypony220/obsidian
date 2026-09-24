---
title: Chrome Mode FSM
tags:
  - pattern
  - fsm
  - state-management
  - frontend
  - tradeoff
date: 2026-04-11
---

# Chrome Mode FSM

moc: [[front-moc]]
next:
- [[flat-vs-orthogonal-state]]
- [[ui-chrome-vocabulary]]
- [[in-memory-navigation-stack]]
- [[stack-vs-step-navigation]]
- [[fsm-closing-state]]
- [[fsm-global-transitions]]
- [[multicast-dispatch]]

---

```
  ┌──────┐  OPEN_MAP_MENU  ┌─────────────────┐  SELECT_MODE  ┌─────────────┐
  │ idle │ ──────────────► │  map-mode-menu  │ ────────────► │ type-picker │
  └──────┘                 └─────────────────┘               └─────────────┘
     ▲                              │                                │ PICK_LOCATION
     │         CANCEL               ▼                                ▼
     │ ◄──────────────── ┌──────────────────┐             ┌──────────────────────┐
     │                   │  editing-filter  │             │   location-picker    │
     │                   └──────────────────┘             │  step: picking       │
     │                            │ APPLY                 │       ↓ PICK_AT_CENTER│
     │ ◄──────────────── ┌──────────────────┐             │  step: confirming    │
                         │  filter-results  │ ──────────► └──────────────────────┘
                         └──────────────────┘  SELECT
```

**Chrome mode** — паттерн описания состояния UI-обвеса через одно дискриминирующее поле `kind` вместо коллекции параллельных флагов. "Chrome" в UI-терминологии — весь обвес вокруг контента (статус-бары, тулбары, футеры, поповеры, навбары). Термин из "window chrome" — рамка окна браузера.

## Идея

Вместо:
```ts
const [showSearchSummaryHeader, setShowSearchSummaryHeader] = useState(false)
const [showBackToResultsCta, setShowBackToResultsCta] = useState(false)
const [showProfileStatusBar, setShowProfileStatusBar] = useState(false)
const [showMapModeMenu, setShowMapModeMenu] = useState(false)
// ... ещё 20+ флагов
```

Одна переменная с discriminated union:
```ts
type ChromeMode =
  | { kind: 'idle' }
  | { kind: 'map-mode-menu' }
  | { kind: 'type-picker'; modeId: MapModeId }
  | { kind: 'location-picker'; step: 'picking' | 'confirming'; draft: Profile }
  | { kind: 'editing-filter'; filters: FilterState }
  | { kind: 'filter-results'; filters: FilterState }
  | { kind: 'filter-results-preview'; filters: FilterState; stack: ProfileView[] }
  | { kind: 'profile'; stack: ProfileView[] }
  | { kind: 'deploy'; draft: Profile; returnTo: ChromeMode | null }
  | { kind: 'auth-required'; reason: string; returnTo: ChromeMode }
```

Переходы — чистая функция `(state, event) → state`:
```ts
function chromeReducer(state: ChromeMode, event: ChromeEvent): ChromeMode {
  switch (state.kind) {
    case 'filter-results':
      if (event.type === 'SELECT_PROFILE')
        return {
          kind: 'filter-results-preview',
          filters: state.filters,
          stack: [{ type: 'preview', profile: event.profile }],
        }
      // ...
  }
}
```

Хром рендерится одной функцией от `mode`:
```tsx
function renderChrome(mode: ChromeMode) {
  switch (mode.kind) {
    case 'filter-results': return <FilterResultsBar filters={mode.filters} />
    case 'filter-results-preview': return <><FilterResultsBar /><ProfilePreviewCard /></>
    // ...
  }
}
```

## Что даёт

- **Взаимное исключение структурно.** Есть одно поле `kind`, двух значений одновременно быть не может. Комбинация "filter-results + profile одновременно" невозможна на уровне типов.
- **Переходы явны.** Из `filter-results-preview` BACK_TO_RESULTS — это буквально один `case` в редьюсере, а не кастомный handler, угадывающий через `closeStack()`.
- **Нет параллельных SOT.** Один reducer вместо десятка `useState` + координатор + context-ы, которые пытаются синхронизироваться через `useEffect`.
- **Typecheck ловит невалидные состояния.** Забыл обработать переход — TS ругается на non-exhaustive switch.
- **Side-effects централизованы.** Все "при входе в kind X запустить Y / при выходе закрыть Z" — в одном месте (эффект на `mode.kind`), а не разбросано по регистрируемым close-коллбэкам.

## Антипаттерн, который лечит

**Множественные SOT + derivation wars.** Когда несколько источников состояния описывают одно и то же ("что сейчас на экране"), они синхронизируются через `useEffect`'ы, которые могут переписывать друг друга:

- `ProfileSelectionContext.state` (стек views)
- `LocationPickerContext.state` (своя state machine)
- `searchState` в компоненте карты
- коллекция `useState` флагов
- `overlayCoordinator.activeOverlay` — "доска объявлений", пытающаяся склеить всё

Правило "при состоянии X должно быть Y" нигде не записано, размазано по десяткам мест. Баги класса "забыли вызвать / зависли в промежуточном состоянии" — типичное следствие.

## Альтернатива: hierarchical statechart (XState)

Иерархические машины с параллельными регионами — мощнее плоского union'а, особенно когда режимов много и они группируются:

```
browsing ─ idle
         └ profile
            └ preview | full | connections | chat
search   ─ editing-filter
         └ results ─ idle | preview
create   ─ type-picker
         └ location-picker ─ picking | confirming
         └ deploy
auth-required  (parallel region, накладывается сверху)
```

**Плюсы:** визуализатор, иерархия даёт compose, параллельные регионы для модальных оверлеев, формальная верификация переходов.

**Минусы:** внешняя зависимость (XState), команда должна привыкнуть к statechart-мышлению, для 5-10 режимов overkill.

**Правило:** плоский discriminated union — пока режимов 10-15 и нет параллельных "слоёв". Statechart — когда начинают нужны иерархия и parallel regions.

## Инкрементальная миграция

Переписывать с нуля обычно нельзя. Порядок:

1. **Shadow mode.** Диспатчим events из существующих handler'ов параллельно, но UI всё ещё читает старое состояние. Asserts на расхождения `computedMode !== legacyMode` ловят пропущенные переходы без регрессий.
2. **Параллельный рендер под feature flag.** Новый chrome читает `ChromeMode`, старый живёт рядом. Визуальное сравнение.
3. **Reducer становится SOT.** Удаляем старые контексты/стейты по одному.
4. **Cleanup.** Снимаем флаг, сносим координатор и `showXxx` флаги.
