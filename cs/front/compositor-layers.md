---
title: Compositor layers и will-change
tags: [concept, browser, performance, animation, frontend]
date: 2026-04-23
---

# Compositor layers

moc: [[front-moc]]
back: [[browser-render-pipeline]]
next:
- [[animation-approaches]]
- [[presence-pattern]]
- [[on-exit-complete-callback]]

---

```
страница
 ├─ корневой слой                           ← общий битмап документа
 │   ├─ обычные элементы (paint сюда)
 │   └─ ...
 ├─ [transform: translateZ(0)]  ← свой слой ← отдельная GPU-текстура
 ├─ [will-change: transform]    ← свой слой
 ├─ <video> / <canvas> / iframe ← свой слой
 └─ position: fixed             ← свой слой

 compositor: transform(layer_N) + opacity(layer_N) → финальный кадр
             без перекраски, без участия main thread
```

**TL;DR:** compositor работает со слоями — отдельными GPU-текстурами. Анимация `transform`/`opacity` уже-существующего слоя идёт без main thread. Но слой должен быть создан **до** старта анимации: если элемент только что вставлен в DOM и main thread занят, браузер может скипнуть `animationstart` и анимация отыграет за 0 мс. `will-change: transform, opacity` промоутит элемент в слой заранее, но стоит VRAM — вешать только на окно анимации через `[data-state]`-квалификатор.

## Что такое слой

Слой (compositor layer) — кусок страницы, нарисованный в **отдельную GPU-текстуру**. Свойства:

- Движется / прозрачится / масштабируется **без перерисовки**. Compositor берёт готовую текстуру и меняет `transform` / `opacity`.
- Живёт в GPU-памяти. Размер ≈ ширина × высота × 4 байта.

Элемент без своего слоя рисуется в слой ближайшего предка-со-слоем (в крайнем случае — в корневой слой документа).

## Когда элемент получает свой слой

Не каждый элемент промоутится автоматически. Триггеры:

- `transform: translateZ(0)` или любая 3D-трансформация
- `will-change: transform` / `will-change: opacity`
- `position: fixed` внутри прокручиваемого контейнера
- `<video>`, `<canvas>`, `<iframe>`
- `filter`, `backdrop-filter`
- `opacity < 1` в некоторых комбинациях
- элемент в разгаре CSS-анимации на compositor-свойстве

Точный список зависит от движка и версии. Chrome DevTools → Rendering → Layer borders показывает фактические слои.

## Почему анимации не попадают на compositor автоматически

Только `transform` и `opacity` можно анимировать целиком на compositor (см. [[browser-render-pipeline]]). Но этого мало — нужен ещё **готовый слой к моменту старта анимации**.

Браузер в момент старта смотрит:

1. Элемент уже имеет свой слой? → отдать анимацию compositor'у.
2. Нет, но свойства анимируются compositor-friendly? → попытаться промоутить сейчас.

Второй путь — хрупкий. Промоушен сам требует Style + Paint (нарисовать элемент в новую текстуру). Если main thread в этот момент уже занят тяжёлой работой, промоушен не успеет, и анимация останется на main thread — со всеми последствиями jank'а.

## Failure mode: скип animationstart под нагрузкой

Сценарий из практики:

1. React коммитит большой diff — монтирует новый футер, перерисовывает сиблингов, обновляет MapGL-слои.
2. Новый футер имеет CSS-анимацию `@keyframes footerIn` на `[data-state="open"]`.
3. Main thread занят Style + Layout + Paint всего коммита.
4. Свежевставленный футер не успевает получить compositor-слой.
5. Браузер решает: кадр не уложимся, **скипаем `animationstart`**, считаем что анимация завершилась мгновенно.
6. Пользователь видит: футер «поп» в финальной позиции без анимации.

Это **оптимизация браузера**, не баг в коде. Лучше показать финальный кадр сразу, чем тормозить весь фрейм ради анимации. Но для UX это провал.

## will-change — явная заявка

```css
.footer {
  will-change: opacity, transform;
}
```

Подсказка браузеру: «этот элемент скоро будет анимироваться — promote в слой заранее, до того как я попрошу анимацию».

Что меняет:

- К моменту старта анимации слой уже существует, Style+Paint кэшированы в текстуре.
- `animationstart` / `transitionstart` не скипается — compositor сразу забирает задачу.
- Main thread может захлёбываться параллельно — анимация футера идёт своим ходом.

## Цена will-change

- Каждый слой = текстура в VRAM: `width × height × 4 байта`. Для полноэкранного элемента на 2560×1440 это ≈ 14 МБ.
- Сотни элементов с `will-change` могут **исчерпать VRAM**, браузер скинет слои обратно — хуже чем без `will-change` (дополнительные расходы на создание/уничтожение).
- `will-change` на элементе, который анимируется раз в минуту, — постоянная трата памяти на простое.

Правило: `will-change` = **временная** заявка, а не постоянный флаг.

## Паттерн: `[data-state]`-квалификатор

Ставим `will-change` только когда Presence-обёртка нацепила `data-state`:

```css
.footer[data-state] {
  will-change: opacity, transform;
}

.footer[data-state="open"]   { opacity: 1; transform: translateY(0); }
.footer[data-state="closed"] { opacity: 0; transform: translateY(100%); }
```

Жизненный цикл:

```
нет атрибута       → нет will-change → нет слоя (VRAM свободна)
data-state="open"  → will-change     → слой создан, анимация на compositor
data-state="closed"→ will-change     → слой держится до animationend
animationend       → Presence unmount → атрибута нет → слой освобождается
```

Это и есть баланс: «щедрый в моменте анимации, скромный в простое». Работает в связке с [[presence-pattern]], который и управляет `data-state`.

## Проверка в DevTools

- Chrome: `⌘⇧P` → `Show layer borders`. Жёлтые рамки — compositor-слои.
- `Performance` tab → запись → видны `Composite Layers`, `Paint`, `Layout` отдельно.
- Если в `animation`-строке видишь `Composited: No` с причиной — значит анимация упала на main thread.

## Концепция шире

Паттерн «закэшировать дорогой результат в отдельный буфер, менять трансформации дёшево» встречается везде где есть rendering/compute:

- **GPU double buffering** — front/back буферы, frame готовится в back, swap дешёвый.
- **CDN / edge cache** — закэшировал ответ, дальше отдаёшь без повторной генерации.
- **Virtual memory pages** — страница загружена в RAM → все последующие чтения быстрые, page fault только при первом доступе.
- **Database materialized views** — дорогой JOIN посчитан один раз, query идёт по готовому результату.

Общее везде: **разделить "создание результата" и "использование результата", закэшировать промежуточное состояние, оплачивать полную цену только при инвалидации**. Compositor-слой — кэш растра для GPU.
