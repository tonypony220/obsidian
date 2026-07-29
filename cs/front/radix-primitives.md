---
title: Radix Primitives — headless React-компоненты с доступностью из коробки
tags: [tool, react, frontend, library]
date: 2026-05-07
---

# Radix Primitives

moc: [[front-moc]]
back: [[component-layers]]
next:
- [[shadcn-ui]]
- [[headless-libraries]]

---

```
  ┌──────────────────────────────────────────────────────┐
  │  Твои стили (Tailwind / CSS modules / vanilla)        │  ← ты пишешь
  ├──────────────────────────────────────────────────────┤
  │  Radix Primitives                                     │  ← бесплатно
  │  ───────────────                                      │
  │  • Slot-архитектура (Root / Trigger / Content / ...)  │
  │  • asChild / Slot для composition                     │
  │  • Presence (mount/unmount + анимация)                │
  │  • Portal                                             │
  │  • Focus trap, ARIA, клавиатура, click outside        │
  │  • data-state="open" | "closed"                       │
  │  • Controlled / uncontrolled из коробки               │
  ├──────────────────────────────────────────────────────┤
  │  React + DOM                                          │
  └──────────────────────────────────────────────────────┘
```

**TL;DR:** Radix Primitives — библиотека headless React-компонентов (Dialog, Popover, DropdownMenu, Tooltip, Select, Accordion, Tabs и т. д.). Даёт **логику и доступность**, не даёт стилей. Канонизировала паттерны [[slot-architecture]], [[as-vs-aschild|asChild]], [[presence-pattern|Presence]], [[portal-and-animation-separation|Portal/Content separation]], `data-state`. shadcn/ui — стилизованные обёртки вокруг Radix, де-факто стандарт 2024+. Тянуть Radix целиком — стандартное решение; писать своё — дорого, нужен веский повод.

## Что это

`@radix-ui/react-*` — набор пакетов, по одному на каждый виджет:

```
@radix-ui/react-dialog
@radix-ui/react-popover
@radix-ui/react-dropdown-menu
@radix-ui/react-tooltip
@radix-ui/react-select
@radix-ui/react-accordion
@radix-ui/react-tabs
@radix-ui/react-toast
...
```

Импортишь только то, что нужно — каждый пакет ~5–20 KB.

## Headless значит

Компоненты дают **поведение и доступность**, но **не дают стилей**. Структура DOM, фокус-менеджмент, ARIA, клавиатурные шорткаты — всё работает. CSS пишешь сам:

```tsx
<Dialog.Root>
  <Dialog.Trigger className="my-button">Open</Dialog.Trigger>
  <Dialog.Portal>
    <Dialog.Overlay className="my-overlay" />
    <Dialog.Content className="my-modal">
      <Dialog.Title>Hello</Dialog.Title>
      <Dialog.Close className="my-close">×</Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

Никаких чужих CSS-классов в результате — твоя дизайн-система не дерётся с библиотечной.

## Что даёт за бесплатно

Это всё, что нужно сделать вручную, если писать самому:

- **Focus trap** в открытом диалоге (Tab не убегает наружу).
- **Возврат фокуса** на trigger при закрытии.
- **Escape** закрывает overlay.
- **Click outside** закрывает.
- **ARIA-атрибуты**: `role="dialog"`, `aria-modal`, `aria-labelledby`, `aria-describedby`.
- **Клавиатурная навигация** в меню/табах/аккордеонах (стрелки, Home/End, type-ahead).
- **Body scroll lock** на время модалки.
- **Portal** для вывода поверх stacking context.
- **`data-state="open" | "closed"`** для CSS-анимаций без флагов.
- **Controlled / uncontrolled** API с консистентными именами (`open` / `defaultOpen`, `onOpenChange`).
- **Safe in SSR**: компоненты не падают при server render.

Самому собрать всё это правильно — недели работы, и всё равно где-то будет дыра в a11y.

## Канонизированные паттерны

Radix не изобрёл, но довёл до индустриального стандарта:

- [[slot-architecture]] — `Root / Trigger / Portal / Overlay / Content / Title / Close`.
- [[as-vs-aschild|asChild]] и `Slot` primitive для композиции с готовыми компонентами.
- [[presence-pattern|Presence]] для mount/unmount-анимаций.
- [[portal-and-animation-separation|Portal / Presence / Content разделение]].
- `data-state="open"|"closed"` атрибут вместо `isOpen` булов в className.
- Controlled / uncontrolled API: каждый компонент — `open` / `defaultOpen` / `onOpenChange`.

Эти паттерны полезны сами по себе, даже если ты не тянешь Radix. Подсмотреть исходники — бесплатное обучение.

## shadcn/ui

[shadcn/ui](https://ui.shadcn.com) — НЕ библиотека в обычном смысле. Это коллекция **готовых стилизованных обёрток** вокруг Radix, которые **копируешь в свой проект** через CLI:

```sh
npx shadcn add dialog
# создаёт файл src/components/ui/dialog.tsx с Radix + Tailwind
```

Файл становится **твоим**, ты его правишь. Не зависишь от обновлений библиотеки. Это решает классическую боль UI-libraries: «нужен мелкий тюнинг → форкать всю библиотеку или писать обёртки».

Стек shadcn = Radix + Tailwind + class-variance-authority. Де-факто стандарт React-проектов 2024+.

## Когда тянуть Radix vs подсматривать

**Тянуть, когда:**
- Проекту нужны стандартные виджеты (Modal, Dropdown, Select, Tooltip, Tabs).
- Доступность важна (b2b, госзаказ, Enterprise).
- Команда не хочет тратить недели на a11y вручную.

**Подсматривать без тяги, когда:**
- Очень маленький проект (1-2 простых модалки) — оверкил.
- Кастомный non-standard виджет, для которого Radix-аналога нет — изучить как у Radix построен ближайший родственник, повторить паттерны.
- Образовательная цель — прочитать `@radix-ui/react-presence` (~150 строк) понятнее любого туториала.

**Не тянуть, когда:**
- React Native — Radix только для DOM. На RN свои библиотеки (Tamagui, gluestack, NativeBase).
- Не React — для Vue/Svelte свои аналоги (Headless UI у Tailwind, Melt UI у Svelte).

## Альтернативы

| Библиотека | Плюсы | Минусы |
|---|---|---|
| **Radix** | самая зрелая, slot-архитектура, экосистема (shadcn) | API многословный, чисто DOM |
| **Headless UI** (Tailwind) | проще API, есть Vue-версия | меньше компонентов, менее гибкий |
| **React Aria** (Adobe) | глубочайшая a11y, hook-based | API сложный, документация плотная |
| **Ariakit** | очень богатый, focus-restore тонко | менее популярен, документация местами слабее |
| **Reach UI** | первопроходец headless React | заброшен, не используй |

Стандартный выбор для нового React-проекта: **Radix + shadcn/ui + Tailwind**. Если a11y критична на уровне Adobe Aria-стандартов — **React Aria**.

## Цена

- **Многословный JSX**: Slot-архитектура требует 5-10 элементов на компонент против 1 в монолите. Привыкаешь.
- **Bundle size**: каждый виджет ~5–20 KB. Не критично, но в сумме набегает на больших проектах с 10+ типами overlay'ев.
- **API learning curve**: понять `Root/Trigger/Content/asChild`, контекстную природу slot'ов, `data-state` — одна-две сессии.
- **Стилей всё ещё писать самому**: headless = свобода + ответственность.

## В твоём проекте

У тебя сейчас собственные `Modal.tsx`, `CenterSheet.tsx`, `useSwipeVertical`. Альтернативы:

1. **Оставить своё** — оправдано, если a11y не приоритет и количество overlay-типов мало.
2. **Перевести overlay'и на Radix Dialog/Popover** — отдельная сессия рефакторинга, выигрыш в a11y и снимет копипасту между Modal/Sheet.
3. **Подсмотреть Presence** ([github.com/radix-ui/primitives/tree/main/packages/react/presence](https://github.com/radix-ui/primitives/tree/main/packages/react/presence)) и реализовать у себя — самый дешёвый способ получить mount/unmount-анимации без полного перехода на Radix.

## Концепция шире

Headless как тренд — не только Radix:

- **Tanstack Query** — headless data fetching: даёт логику кэша/инвалидации, UI рисуешь сам. Альтернатива — Apollo с готовыми компонентами.
- **Tanstack Table** — headless table: даёт sort/filter/pagination logic, рендер ячеек твой. Альтернатива — AG Grid с готовым UI.
- **react-hook-form** — headless forms: валидация и state, поля рисуешь сам.
- **Headless CMS** (Strapi, Sanity) — хранение и API без preset-фронтенда.
- **В компиляторах**: front-end (parsing AST) и back-end (codegen) — разделение «логика отдельно, представление отдельно». LLVM строится на этом принципе.

Общее везде: **разделить логику от представления, чтобы каждое можно было менять независимо**. Цена — нужно собирать composition самому; выгода — не дерёшься с чужими дефолтами и стилями.
