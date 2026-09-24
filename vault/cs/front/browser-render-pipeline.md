---
title: Browser render pipeline — Style/Layout/Paint/Composite
tags: [concept, browser, performance, frontend]
date: 2026-04-23
---

# Browser render pipeline

moc: [[front-moc]]
next:
- [[compositor-layers]]
- [[animation-approaches]]
- [[presence-pattern]]
- [[request-animation-frame]]

---

```
JS/события
    │
    ▼
 ┌────────┐   ┌────────┐   ┌───────┐   ┌───────────┐
 │ Style  │──▶│ Layout │──▶│ Paint │──▶│ Composite │──▶ пиксели
 └────────┘   └────────┘   └───────┘   └───────────┘
 ────────────────main thread─────────── │ compositor
                                        │   thread
 всё должно уложиться в 16.6 мс/кадр при 60 Hz
 не уложился → jank (пропуск кадра)
```

**TL;DR:** браузер рисует каждый кадр через pipeline `Style → Layout → Paint → Composite`. Первые три живут на main thread (вместе с твоим JS), последний — на compositor thread. Свойство `transform` и `opacity` анимируются целиком на compositor и не блокируются тяжёлым JS. Всё остальное (`top`, `width`, `background-color`, …) требует пересчёта на main thread каждый кадр и встаёт, когда main thread занят.

## Стадии

- **Style** — для каждого узла посчитать, какие CSS-правила применяются.
- **Layout (reflow)** — посчитать геометрию: где элемент, какого размера.
- **Paint** — закрасить пиксели каждого элемента в растровый битмап.
- **Composite** — собрать битмапы в один финальный кадр, отправить в GPU.

Бюджет одного кадра: 16.6 мс при 60 Hz (1000/60). Не уложились — пользователь видит jank: рывок, подвисание, пропущенный кадр.

## Main thread vs compositor thread

В современных движках стадии разнесены по двум потокам:

| Поток | Что делает |
|---|---|
| **Main thread** | твой JS, Style, Layout, Paint |
| **Compositor thread** | берёт уже готовые слои-битмапы, склеивает в кадр |

Compositor не зависит от JS. Пока он работает с готовыми слоями, main thread может быть занят чем угодно — следующий кадр всё равно нарисуется.

## Пример: спиннер под тяжёлым JS

```js
for (let i = 0; i < 1e9; i++) {}  // main thread заблокирован на секунды
```

Спиннер через `transform: rotate(…deg)` — **продолжает крутиться плавно**. Анимация живёт на compositor, ему main thread не нужен.

Тот же спиннер через `top` / `left` — **встаёт колом**. Эти свойства требуют Layout, а Layout живёт на main thread.

## Какие свойства на каком потоке

- **Compositor-friendly (без main thread):** `transform` (translate, rotate, scale, skew), `opacity`.
- **Layout-triggering:** `width`, `height`, `top`, `left`, `margin`, `padding`, `font-size`, …
- **Paint-triggering:** `background-color`, `color`, `box-shadow`, `border-radius`, …

Layout- и paint-триггерящие свойства проходят весь pipeline на main thread **каждый кадр** анимации. Если main thread занят — анимация встаёт или мигает.

Это же причина, по которой CSS-анимации/переходы на `transform` и `opacity` считаются "дешёвыми", а на `width`/`top` — "дорогими".

## Следствие для анимаций

- Анимации UI-хрома (модалки, футеры, поповеры) пиши через `transform` + `opacity` — они не делят main thread с JS.
- Но даже `transform`-анимация может сорваться, если элемент **стартует одновременно с тяжёлым React-коммитом** и не успел получить compositor-слой. См. [[compositor-layers]].

## Концепция шире

Pipeline «декларативное описание → промежуточные стадии → финальный вывод» — общий паттерн rendering/compute engines:

- **GPU rendering pipeline** — vertex → tessellation → geometry → rasterization → fragment. Те же стадии, те же трейдоффы «что на CPU, что на GPU».
- **Compiler pipeline** — lex → parse → typecheck → IR → codegen. Разные стадии на разных ресурсах/потоках.
- **CI/CD** — lint → test → build → deploy. Раннее ломаем дёшево, позднее — дорого.
- **SQL query execution** — parse → plan → optimize → execute. Планировщик отдельно от исполнителя.

Общее везде: **фиксированный pipeline стадий + разделение по потокам/ресурсам даёт параллелизм и позволяет менять что-то на поздней стадии без пересчёта ранних**.
