---
title: Component layers — Primitive / Behavior / Container
tags: [concept, pattern, react, frontend]
date: 2026-04-20
---

# Component Layers — Primitive / Behavior / Container

moc: [[front-moc]]
next:
- [[forward-ref]] — как в primitive прокинуть ref к DOM
- [[polymorphic-as-prop]] — один компонент, любой semantic HTML
- [[controlled-vs-uncontrolled]] — кто владеет state компонента
- [[presence-pattern]] — удержание в DOM во время анимации
- [[rtl-wiring-test]] — тест правильности связки Container → hook → builder

---

```
      data / context         tokens
            │                   │
            ▼                   ▼
     ┌──────────────┐    ┌──────────────┐
     │  Container   │───▶│  Primitive   │───▶ DOM
     │ (знает фичу) │    │ (знает токены)│
     └──────────────┘    └──────────────┘
            ▲
            │ state / handlers
     ┌──────────────┐
     │   Behavior   │
     │  (hook, без  │
     │     JSX)     │
     └──────────────┘
```

**TL;DR:** Три слоя по ответственности и частоте изменений: **Primitive** рисует DOM и знает только токены; **Behavior** — хук без JSX с механикой; **Container** склеивает первые два и знает про фичу.

## Слои

**Primitive** — чистый рендер, `props → DOM`. Знает дизайн-токены (`spacing`, `colors`, `radius`), не знает ничего про проект и его фичи. Живёт годами без правок. Пример: `Box`, `Stack`, `Text`.

**Behavior** — логика без JSX. Хук, возвращает `{ state, handlers }`. Не знает, на что его навесят. Пример: `useSwipeVertical` → `{ onTouchStart, onTouchEnd }`.

**Container** — «умный» компонент. Знает контексты, роуты, данные. Сам `<div>` не пишет — складывает композицию из primitives + behavior-хуков. Пример: `FullScreenPopup`.

## Почему разделять

**Разная частота изменений**:
- Primitive пишется один раз, часть дизайн-языка. Меняется редко.
- Container переписывается постоянно — новые фичи, API, роуты.

Смешаешь — каждое изменение фичи трогает разметку примитива. Через полгода никто не рискует править `Box`, потому что от него зависят 40 контейнеров.

## Где живут стили

- Primitive знает **токены**, не знает **фичи**. `padding: 16px` в primitive → плохо; `padding: spacing[p]` → хорошо.
- Специфика фичи — **обёрткой**, а не `variant` props:
  ```tsx
  const ChatBubble = (props) => (
    <Box p="sm" radius="lg" bg="surfaceSoft" {...props} />
  );
  ```
- Граница: токены = дизайн-язык (глобально), вариант под фичу = container-слой.

## Маркеры смешения слоёв

- В primitive появился `fetch()`, `useContext(SomethingAppSpecific)` — primitive протёк в container.
- Behavior-хук начал возвращать JSX — behavior протёк в primitive.
- В контейнере `<div style={{ padding: 16 }}>` вместо `<Box p="lg">` — container полез в дизайн-систему мимо примитивов.

## Концепция шире

Та же структура встречается не только в React:
- **Unix**: coreutils (primitive) / pipes (behavior) / shell-скрипты (container).
- **Backend**: value objects (primitive) / domain services (behavior) / use-cases (container).
- **CSS**: utility-классы (primitive) / компонентные классы (container) — BEM и Tailwind строят на этой границе.
- **ML**: слой нейросети (primitive) / loss/optimizer (behavior) / training loop (container).

Общее везде: **разделение по частоте изменений и области знания** — стабильные универсальные блоки внизу, изменчивая бизнес-логика наверху, склейка — отдельной сущностью.
