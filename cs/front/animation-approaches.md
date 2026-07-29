---
title: Подходы к анимации — от CSS transitions до Canvas
tags: [concept, comparison, animation, frontend]
date: 2026-04-24
---

# Animation approaches

moc: [[front-moc]]
back: [[browser-render-pipeline]]
next:
- [[compositor-layers]]
- [[presence-pattern]]
- [[request-animation-frame]]

---

```
дёшево, декларативно                                    дорого, императивно
──────────────────────────────────────────────────────────────────────▶

 transitions   @keyframes   WAAPI   FLIP   View Transitions   Framer   Canvas
                                      │        API             Motion    SVG
     A→B       много шагов    рантайм │       shared          layoutId   WebGL
  одно         фиксированных  параметры│      element         авто-FLIP  физика
 свойство       шагов          из данных│       нативно                  частицы
                                         │
                               измеряешь до/после,
                               инвертируешь transform
```

**TL;DR:** у анимации в вебе 7 уровней. Слева — дёшево и декларативно (CSS), справа — дорого и императивно (Canvas). Правило выбора: самый левый уровень, который выражает задачу. `transitions` для on/off. `@keyframes` для multi-stage. WAAPI когда параметры считаются в runtime. FLIP для layout-морфинга без дёрганий. View Transitions — нативный shared-element (но Safari 18+, в RN нет). Framer Motion — `layoutId` делает FLIP автоматически, но +50 KB и в RN мигрирует плохо. Canvas/SVG/WebGL — когда частицы и physics. В RN правильный слой — `Reanimated` (UI thread ≈ compositor thread браузера).

## 1. CSS transitions

Плавный переход между двумя состояниями по одному свойству.

```css
.footer { opacity: 0; transition: opacity 200ms; }
.footer[data-state="open"] { opacity: 1; }
```

- Только start → end, без промежуточных кадров.
- Compositor-friendly если свойство из списка (см. [[compositor-layers]]).
- Триггер — смена класса/атрибута/inline-стиля.

**Где применять:** on/off-состояния (hover, open/closed), one-shot переходы. 80% UI-анимаций.

## 2. CSS @keyframes

Многошаговая последовательность с процентными контрольными точками.

```css
@keyframes bounce {
  0%   { transform: translateY(-100%); }
  60%  { transform: translateY(10%); }
  100% { transform: translateY(0); }
}
```

- Нелинейный путь, bounce, multi-stage.
- Параметры — **фиксированные в CSS** на билд-тайме.
- Та же compositor-история что у transitions.

**Где применять:** презентационные эффекты с заранее известной траекторией (spinner, shimmer, pulse, attention-seeker).

## 3. WAAPI (Web Animations API)

То же что `@keyframes`, но параметры **вычислимы в runtime**:

```ts
element.animate(
  [
    { transform: `translate(${startX}px, ${startY}px)` },
    { transform: `translate(${endX}px, ${endY}px)` },
  ],
  { duration: 300, easing: 'cubic-bezier(...)' }
);
```

- Возвращает `Animation` объект — можно `pause()`, `reverse()`, `finished` Promise.
- Keyframes в JS-объектах, можно считать из данных (координаты маркера, размер экрана).
- Тот же compositor-путь если анимируешь `transform`/`opacity`.

**Где применять:** анимации с runtime-параметрами (карта, drag-anywhere, dynamic layouts). Промежуточный слой между CSS и полноценными motion-либами.

## 4. FLIP (First-Last-Invert-Play)

Паттерн для морфинга «элемент был тут → стал там» без дёрганий.

```
1. First    — измеряешь позицию/размер до изменения (getBoundingClientRect)
2. Last     — применяешь изменение, измеряешь новую позицию
3. Invert   — навешиваешь transform, который компенсирует разницу (элемент
              визуально остался на старом месте, но в DOM уже на новом)
4. Play     — анимируешь transform к identity (transform: none)
```

- **Только `transform`/`opacity`**, значит compositor-friendly даже при смене layout.
- Работает на любом DOM-изменении — reorder списка, смена колонок, переход между страницами.
- Не требует либ, но писать руками утомительно.

**Где применять:** shared element transitions, list reorder, layout-морфинг. Фундамент `layoutId` в Framer Motion и `View Transitions API`.

## 5. View Transitions API

Нативный браузерный API специально для FLIP-подобных переходов:

```ts
document.startViewTransition(() => {
  // любая мутация DOM
  setRoute('/profile/42');
});
```

- Браузер сам делает снимок до, снимок после, FLIP'ает между ними.
- Для shared elements — `view-transition-name: hero` на обоих (до и после).
- Chrome/Edge — ок. Safari 18+ (появился недавно). Firefox — в работе.
- **В React Native нет и не будет** — это DOM API.

**Где применять:** view-to-view переходы на web-only проектах, где Safari 18+ устраивает. На кросс-платформе (включая RN) — не закладываться.

## 6. Motion-либы (Framer Motion, Motion One)

Высокоуровневая обёртка поверх WAAPI/FLIP:

```tsx
<motion.div layoutId="hero" animate={{ x: 100, opacity: 1 }} />
```

- `layoutId` — shared layout transitions автоматически (то же что View Transitions, но кросс-бразурно).
- Spring-физика, жесты, drag/gesture API.
- **Bundle size:** Framer Motion ≈ 50 KB gz. Motion One — ≈ 3-5 KB, но проще по возможностям.
- **RN:** `framer-motion` не поддерживается, есть `framer-motion-3d` и `moti` (построен на Reanimated) — другой API.

**Где применять:** продуктовые UI с частыми layout-морфингами. На кросс-платформе — заранее планируй, что RN-сторона поедет на `Reanimated`/`moti`, а не на Framer.

## 7. Canvas / SVG / WebGL

Императивный рендер в свою поверхность, минуя DOM-layout:

- **Canvas 2D** — частицы, trail-эффекты, progress/meter с анимацией линий.
- **SVG + SMIL/JS** — векторная анимация, path morph, иконки.
- **WebGL / WebGPU** — physics, large-scale particles, 3D.

- Обходит весь render pipeline ([[browser-render-pipeline]]) — ты сам управляешь тем, что попадёт в GPU.
- Не переиспользуется с accessibility, hit-testing, z-index'ом DOM.

**Где применять:** частицы, physics simulation, визуализации. Для обычного UI — избыточно.

## React Native — что портируется

Принципиально: **нет DOM, нет CSS**. Система анимаций — свой `Animated` API или `react-native-reanimated` (новый стандарт).

Ключевая параллель с браузером: **UI thread в RN ≈ compositor thread в браузере.** Reanimated гоняет анимации на UI thread, JS thread может быть заблокирован — анимация продолжает играть. Это прямой аналог compositor-friendly свойств ([[compositor-layers]]).

| Уровень | В React Native |
|---|---|
| CSS transitions | Нет. Заменитель — `Animated.timing` / Reanimated `withTiming`. |
| CSS `@keyframes` | Нет. Через последовательность `withSequence(withTiming(...), withTiming(...))`. |
| WAAPI | Нет. Reanimated `useSharedValue` + worklets — ближайший аналог. |
| FLIP | Работает, но `getBoundingClientRect` нет. Используй `measure()` / `onLayout`. |
| View Transitions API | Нет и не будет. |
| Framer Motion | Нет. Эквивалент — `moti` (построен на Reanimated) или напрямую Reanimated. |
| Canvas / SVG / WebGL | `react-native-skia` (Skia), `react-native-svg`, `expo-gl` для WebGL. |

**Выбор для кросс-платформенного проекта:**
- На web: CSS transitions + Presence ([[presence-pattern]]) для on/off, FLIP руками для shared-element, мотион-либа только если оправдана bundle-ценой.
- На RN: Reanimated как базовый слой для всего. `moti` если хочется декларативного API как у Framer.
- **Не закладывайся на View Transitions**, если RN-ветка обязательна.
- **Дизайн анимаций общий** (тайминги, easing, траектории), **исполнение — раздельное** (CSS на web, Reanimated на RN). Общий спец описывает **что анимируется**, каждая платформа решает **как**.

## Правило выбора уровня

Сверху вниз — чем выше, тем дешевле:

1. Можно выразить через CSS-переключение атрибута? → **transitions**.
2. Нужны промежуточные ключевые кадры с **фиксированными** значениями? → **@keyframes**.
3. Параметры зависят от runtime-данных? → **WAAPI**.
4. Элемент «переезжает» между позициями в DOM? → **FLIP** (или View Transitions на web-only, или `layoutId` если Framer уже в бандле).
5. Много одновременных физических движений, частиц? → **Canvas/WebGL**.

Не брать Framer Motion «на всякий случай» — 50 KB это налог на каждый старт приложения.

## Концепция шире

Спектр «декларативное → императивное, дешёвое → дорогое» встречается везде где есть rendering:

- **SQL vs хранимые процедуры vs raw cursor** — декларативный запрос дешевле императивного обхода.
- **CSS Grid vs Flexbox vs absolute positioning** — чем декларативнее, тем больше оптимизаций браузер может сделать.
- **React vs imperative DOM manipulation** — декларативное описание состояния vs ручные `appendChild`.
- **Declarative IaC (Terraform) vs imperative scripting (bash)** — декларация желаемого vs рецепт как туда прийти.

Общее везде: **декларативная форма даёт исполнителю (браузеру, БД, фреймворку) свободу оптимизаций — он знает намерение, а не пошаговый рецепт**. Цена — потеря гибкости для нетиповых случаев, когда приходится спускаться на императивный уровень.
