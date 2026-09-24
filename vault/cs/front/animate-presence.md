---
title: AnimatePresence — JS-driven Presence из framer-motion
tags: [tool, react, frontend, animation]
date: 2026-04-23
---

# AnimatePresence (framer-motion)

moc: [[front-moc]]
back: [[presence-pattern]]
next:
- [[on-exit-complete-callback]]
- [[shared-element-transitions]]

---

```
  mount                         unmount
  ─────                         ───────

  Radix Presence:               Radix Presence:
   DOM появился                  state=open
   └─ state=closed               └─ state=closed
      └─ rAF                        └─ CSS transition (200ms)
         └─ state=open                 └─ transitionend
            └─ CSS transition             └─ unmount

  framer AnimatePresence:       framer AnimatePresence:
   DOM появился                  animate={...}
   └─ initial={...}              └─ exit={...}
      └─ JS rAF тики                └─ JS rAF тики
         └─ animate={...}              └─ onComplete
                                          └─ unmount
```

**TL;DR:** `<AnimatePresence>` — Presence-обёртка из framer-motion. Делает то же, что [[presence-pattern|Radix Presence]] (держит узел в DOM до конца exit-анимации), но анимирует **сама на JS** через `requestAnimationFrame`. Контракт с детьми — пропсы `initial`/`animate`/`exit` на `motion.*`-компонентах, а не CSS+`data-state`. Цена — ~50 KB бандла и JS на каждый кадр; выгода — springs, layoutId, gestures, stagger orchestration.

## API

```tsx
import { AnimatePresence, motion } from 'framer-motion';

<AnimatePresence>
  {isOpen && (
    <motion.div
      initial={{ opacity: 0, scale: 0.9 }}   // стартовое состояние (до видимого mount)
      animate={{ opacity: 1, scale: 1 }}     // целевое (после mount)
      exit={{ opacity: 0, scale: 0.9 }}      // при unmount
      transition={{ duration: 0.2 }}
    >
      Content
    </motion.div>
  )}
</AnimatePresence>
```

Пошагово при `isOpen: false → true`:
1. React монтирует `<motion.div>`.
2. framer применяет `initial` как стартовый стиль.
3. На следующий кадр интерполирует к `animate` через JS (rAF).
4. Через 200ms — анимация завершена.

При `isOpen: true → false`:
1. React хочет размонтировать.
2. `AnimatePresence` перехватывает: «сначала exit».
3. framer интерполирует от `animate` к `exit`.
4. По завершении — `AnimatePresence` разрешает unmount → React убирает узел.

## Отличие от Radix Presence

| | Radix `<Presence>` | framer `<AnimatePresence>` |
|---|---|---|
| Кто анимирует | браузер (CSS transitions/animations) | JS (интерполяция на rAF) |
| Контракт с детьми | `data-state="open"/"closed"` атрибут | пропсы `initial`/`animate`/`exit` |
| Тип компонента детей | любой DOM-элемент | только `motion.*` или `motion(MyComponent)` |
| Сигнал «exit готов» | `transitionend`/`animationend` event | внутренний колбек framer |
| Размер | ~100 строк, 0 KB зависимостей | ~50 KB (framer-motion) |
| CPU нагрузка | почти ноль (compositor / GPU) | каждый кадр = JS-расчёт |
| Возможности | всё, что умеет CSS | всё CSS + springs + layout + gestures |

## Что framer даёт сверху, чего нет у CSS

### 1. Spring-физика

```tsx
transition={{ type: 'spring', stiffness: 400, damping: 30 }}
```
Натуральное «пружинящее» приземление с overshoot и затуханием. CSS этого нативно не умеет — приходится подделывать кубическими безье, и всё равно неточно.

### 2. `layoutId` — shared element transitions

```tsx
{/* список */}
<motion.img layoutId="photo-42" src="..." />

{/* детальная страница, другой компонент / маршрут */}
<motion.img layoutId="photo-42" src="..." />
```
framer обнаруживает, что элемент с тем же `layoutId` появился в новом месте → плавно «перелетает» туда, интерполируя position/size/scale. Аналог shared element transitions из Android. В CSS — десятки строк ручного кода с FLIP-техникой.

### 3. Gesture integration

```tsx
<motion.div
  drag="x"
  dragConstraints={{ left: 0, right: 200 }}
  whileHover={{ scale: 1.05 }}
  whileTap={{ scale: 0.95 }}
/>
```
Drag, hover, tap — встроены и интегрируются с анимациями. CSS даёт `:hover`, но не drag и не плавные трансформации между состояниями.

### 4. Variants и orchestration

```tsx
const list = {
  show: { transition: { staggerChildren: 0.05 } }
};
const item = {
  hidden: { opacity: 0, y: 10 },
  show:   { opacity: 1, y: 0 }
};

<motion.ul variants={list} initial="hidden" animate="show">
  {items.map(i => <motion.li key={i.id} variants={item}>{i.text}</motion.li>)}
</motion.ul>
```
Список появляется по очереди с задержкой 50ms между элементами. Без библиотеки — нетривиально, нужно вручную считать delays.

## Тонкости

### `motion.div` ≠ `<div>`

`motion.div` — обёртка над `<div>`, регистрирующая узел в системе framer. Пропсы `initial/animate/exit` работают **только** на `motion.*` компонентах. Передашь их на обычный `<div>` — React выкинет варнинг про неизвестные атрибуты.

Для своих компонентов: `const MotionMyThing = motion(MyThing)` — но `MyThing` обязан принимать ref через `forwardRef` ([[forward-ref]]), иначе framer не сможет навешивать стили.

### `mode` в AnimatePresence

```tsx
<AnimatePresence mode="wait">      // ждать exit одного → mount следующего
<AnimatePresence mode="popLayout"> // exit-элемент покидает layout сразу
<AnimatePresence>                  // (default sync) — exit и mount одновременно
```
Важно для page transitions: `wait` даёт чистую последовательность «старая страница ушла → новая пришла», `sync` накладывает их (хорошо для cross-fade).

### `key` обязателен для смены детей

```tsx
<AnimatePresence>
  <motion.div key={currentPage} ... />
</AnimatePresence>
```
Без уникального `key` AnimatePresence не понимает, что один ребёнок «ушёл», а другой «пришёл» — ему важно отличать identity.

## Когда что брать

- **Простые fade/slide/scale-переходы** (модалки, тосты, dropdown) → Radix `<Presence>` + CSS. Дешевле, быстрее, 0 KB.
- **Springs, layoutId, drag, stagger-списки** → framer `<AnimatePresence>` + `motion`. ~50 KB бандла, но API окупает себя.
- **Смешанный подход** — большинство overlay'ев на Radix, сложные page-transitions и shared elements на framer. Стандартная практика крупных проектов.

## Суть в одной строке

`<Presence>` (Radix) = **контракт «живи до transitionend»**, анимация в CSS.
`<AnimatePresence>` (framer) = **контракт «живи до окончания моей JS-анимации»**, анимация декларируется пропсами.

Идея одна — удержать узел в DOM на время exit. Движок и API разные.
