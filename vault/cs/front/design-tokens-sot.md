---
title: Design tokens — runtime vs CSS как SOT
tags: [concept, tradeoff, react, frontend, css]
date: 2026-05-07
---

# Design tokens — runtime vs CSS

moc: [[front-moc]]
back: [[component-layers]]
next:
- [[data-layer-sot]]
- [[css-custom-properties]]

---

```
  Runtime SOT (значения в JS)              CSS SOT (значения в CSS)

  tokens.ts                                tokens.css
  ─────────                                ──────────
  spacing.lg = 16                          --spacing-lg: 16px
       │                                         │
       │ import { spacing }                      │ cascade / var()
       ▼                                         ▼
  Box.tsx                                  .box { padding: var(--spacing-lg) }
  <div style={{                                  │
    padding: spacing[p]   ─── рендерится ──▶    │
  }}/>                                           ▼
       │                                  DOM: <div class="box">
       ▼                                  computed: padding: 16px
  DOM: <div style="padding:16px">

  ▲ значения видны JS,                    ▲ значения скрыты от JS,
    inline в каждом узле                    JS видит только имена/классы
```

**TL;DR:** Дизайн-токены могут жить в **JS-модуле** (runtime SOT) или в **CSS** (классы / CSS-переменные). Runtime — простой, SSR-friendly, но смена темы = ре-рендер всего дерева. CSS — почти бесплатная темизация (через `data-theme` + `var()`), GPU-композит, но сложнее с derived values. Современный стандарт — гибрид: значения в CSS-переменных, TS-фасад над именами для типизации.

## Что такое токены

Именованные значения дизайн-системы:

```
spacing.sm = 8        colors.surface = #fafafa     radius.md = 8
spacing.md = 12       colors.surfaceSoft = #f0f0f0  radius.lg = 12
spacing.lg = 16       colors.accent = #0066ff       radius.full = 9999
```

Все примитивы ([[component-layers|Box, Stack, Text]]) ссылаются на токены, не на сырые числа. Вопрос — **где физически живёт `16`**.

## Вариант А: Runtime (TS-модуль)

```ts
// design/tokens.ts
export const spacing = { sm: 8, md: 12, lg: 16, xl: 24 };
export const colors  = { surface: '#fafafa', accent: '#0066ff' };
```

```tsx
// Box.tsx
import { spacing, colors } from 'design/tokens';

const Box = ({ p, bg }) => (
  <div style={{ padding: spacing[p], background: colors[bg] }} />
);
```

JS читает `spacing.lg` на каждом рендере → проставляет inline-style `style="padding:16px"`. **Истина — в TS-модуле.**

## Вариант Б: CSS (классы или переменные)

Через классы:

```css
.p-lg { padding: 16px; }
.bg-surface { background: #fafafa; }
```
```tsx
<div className={`p-${p} bg-${bg}`} />
```

Через CSS-переменные (cascadable):

```css
:root {
  --spacing-lg: 16px;
  --color-surface: #fafafa;
}
.box { padding: var(--spacing-lg); background: var(--color-surface); }
```

JS значений `16` / `#fafafa` **не видит**. Знает только имена. **Истина — в CSS.**

## Сравнение

| Аспект | Runtime (JS) | CSS |
|---|---|---|
| Темизация | новый объект → ре-рендер всего дерева | свитч `data-theme` на root, 0 React-ререндеров |
| SSR | inline в HTML, работает сразу | нужен preload CSS, иначе FOUC |
| Derived values | легко (`spacing.lg * 2`) | сложнее (`calc(var(--spacing-lg) * 2)`) |
| Type safety | нативный TS на значения | TS только на имена токенов |
| Где лежит в DOM | inline `style="..."` | computed через классы/переменные |
| Анимация значения | плохо (rerender React на каждом кадре) | хорошо (CSS transition без JS) |
| Перформанс ререндера | inline-style — дёшево, но не free | className-смена — почти free |
| Где в бандле | в JS | в CSS (кэшируется браузером отдельно) |

## Главный pain-point — темизация

**Runtime:**
```tsx
const theme = useContext(ThemeContext);
<div style={{ background: theme.colors.surface }} />
```
Смена темы → новый context value → ре-рендер **всего дерева**. На крупном приложении 1–2 секунды JS-работы.

**CSS:**
```css
:root            { --color-surface: #fafafa; }
[data-theme="dark"] { --color-surface: #1a1a1a; }
```
```tsx
<html data-theme={theme}>...</html>
```
JS меняет один атрибут — браузер пересчитывает CSS-переменные сам. **Ноль ререндеров React, ноль JS-работы.** По сути бесплатно.

## Современный стандарт — гибрид

CSS как SOT значений + JS как типизированный фасад над именами:

```css
/* tokens.css */
:root {
  --spacing-sm: 8px;
  --spacing-md: 12px;
  --spacing-lg: 16px;
  --color-surface: #fafafa;
}
[data-theme="dark"] {
  --color-surface: #1a1a1a;
}
```

```ts
// tokens.ts — TS-фасад
export const spacing = {
  sm: 'var(--spacing-sm)',
  md: 'var(--spacing-md)',
  lg: 'var(--spacing-lg)',
} as const;

export const colors = {
  surface: 'var(--color-surface)',
} as const;
```

```tsx
<div style={{ padding: spacing.lg, background: colors.surface }} />
```

Получается:
- **Real values в CSS** → темизация бесплатна, доступен GPU-композит, кэш браузера.
- **TS-типизация на именах** → IDE автокомплит, нет magic strings в JSX.
- **SSR работает** (CSS-переменные cascadят сразу).
- **JS не таскает магические числа** → бандл чище.

## Готовые решения

- **Tailwind** — CSS-классы под капотом, темизация через `dark:` и `data-theme`. Самое распространённое решение 2024+.
- **vanilla-extract / Panda CSS** — пишешь стили в TS, компилируется в статический CSS на build-time. CSS-переменные генерятся автоматически. Лучшие гибриды.
- **CSS Modules + кастомные tokens.css** — ручной вариант без библиотек.
- **Stitches / styled-components / Emotion** (runtime CSS-in-JS) — устарели, проблемы с SSR и перформансом. В новых проектах не использовать.

## Когда runtime (JS-SOT) оправдан

- **Сильно derived стили**: `padding: itemIndex * 4 + 8` — в CSS неудобно.
- **Анимации значений по JS-логике**: значение зависит от `scrollY`, drag-position, физики — JS уже считает, дубль в CSS бесполезен.
- **Canvas / WebGL сцены** — CSS не участвует.
- **Прототипы** — быстрее накидать TS-объект, чем настраивать build-pipeline для CSS.

В обычных UI-компонентах **CSS-SOT — выбор по умолчанию**.

## Связь с другими паттернами

- **Tokens живут в правильном слое примитива** — [[component-layers]] (primitive знает токены, не знает фичи).
- **Темизация без ререндера** = тот же принцип, что [[fix-left-to-right]] — двигай дёшево-верифицируемые решения вниз стека.
- **`var()` cascadят** — через `data-state="open"` модальная карточка может **переопределить** конкретные переменные в открытом состоянии. Связь с [[presence-pattern]].

## Концепция шире

Та же дилемма «где живёт значение» встречается всюду, где есть source-of-truth решения:

- **App config**: env vars (runtime) vs `config.ts` (compile-time) vs remote config service (network) — каждый уровень даёт разную скорость переключения и разную стоимость деплоя.
- **i18n**: словари в JS-bundle (быстро, но bundle растёт) vs lazy-loaded JSON (бандл маленький, есть лаг при смене языка) vs CDN-files (можно обновить без релиза).
- **Feature flags**: hardcoded в коде vs runtime SDK (LaunchDarkly) vs CSS-only (`@media (prefers-reduced-motion)` и т. п.).
- **Кэширование**: значения в Redis vs в БД vs derived из других значений (computed columns).

Общее везде: **выбор уровня хранения определяет стоимость изменения и масштаб эффекта**. Чем «выше» в стеке (runtime) — тем дороже массовая правка; чем «ниже» (compile, CSS) — тем дешевле и атомарнее.
