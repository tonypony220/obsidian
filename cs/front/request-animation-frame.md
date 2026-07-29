---
title: requestAnimationFrame — синхронизация JS с кадром экрана
tags: [concept, browser, animation, performance, frontend]
date: 2026-05-14
---

# requestAnimationFrame (RAF)

moc: [[front-moc]]
back: [[browser-render-pipeline]]
next:
- [[animation-approaches]]
- [[compositor-layers]]
- [[presence-pattern]]

---

```
браузер:  │── tick ──│── tick ──│── tick ──│── tick ──│   60 Hz
          ▼          ▼          ▼          ▼          ▼
RAF:    tick(t0)   tick(t1)   tick(t2)   tick(t3)   tick(t4)
          │          │          │          │
          ▼          ▼          ▼          ▼
       мутируешь  мутируешь  мутируешь  мутируешь
       state      state      state      state
                  │ браузер сразу за callback'ом
                  ▼ делает Style/Layout/Paint/Composite
                  один paint в конце кадра

setInterval(fn, 16):
   t=0    t=16   t=32   t=48   t=64   ...   дрейф, не привязан к vsync
   │      │      │      │      │           может выстрелить дважды
   ▼      ▼      ▼      ▼      ▼           между кадрами → впустую
```

**TL;DR:** `requestAnimationFrame(cb)` просит браузер позвать `cb(timestamp)` **прямо перед следующим кадром**. Это синхронизация с vsync дисплея (60 Hz → ~16.6 мс, 120 Hz ProMotion → ~8.3 мс). В отличие от `setInterval` — не дрейфует, паузится во вкладке в фоне, группируется в один paint за кадр. Стандартный API для всего, что двигает пиксели по экрану из JS: императивные анимации, drag, scroll-driven эффекты, цикл `setPaintProperty` на canvas/WebGL/MapLibre. Идиома — рекурсивный `tick(now) → … → requestAnimationFrame(tick)`, отменяется через `cancelAnimationFrame(id)`. Гейтить запуск условием (если анимировать нечего — не крутить цикл), иначе сжигает батарею.

## Контракт API

```ts
const id = requestAnimationFrame((timestamp: DOMHighResTimeStamp) => {
  // вызывается перед следующим кадром
  // timestamp == performance.now() на момент начала кадра
});

cancelAnimationFrame(id);  // отменить запланированный callback
```

- Один `requestAnimationFrame` = **один** будущий вызов. Не повторяется автоматически — нужно перезапросить внутри callback'а.
- `timestamp` — high-resolution (доли мс), одинаковый для всех RAF-callback'ов одного кадра. Использовать его, а не `Date.now()` / `performance.now()` внутри — это даёт точную фазу анимации без рассинхрона между параллельными RAF'ами.

## Зачем не `setInterval(fn, 16)`

| Свойство | `setInterval(fn, 16)` | `requestAnimationFrame` |
|---|---|---|
| Привязка к vsync | нет, дрейф | да, синхронно с refresh rate |
| Refresh rate монитора | игнорирует (всегда 16 мс) | подстраивается (60/90/120/144 Hz) |
| Вкладка в фоне / окно свёрнуто | продолжает тикать | браузер ставит на паузу |
| Несколько callback'ов в кадре | каждый дёргает Style/Layout отдельно | браузер группирует → один paint |
| Когда срабатывает | в любой момент между кадрами | строго перед началом следующего кадра |

Главная цена `setInterval`: callback может выстрелить **дважды между двумя соседними кадрами** (если main thread задержался) — оба раза пересчитал style, второй раз впустую, потому что пользователь увидит только последнее значение. RAF гарантирует «один callback ↔ один кадр».

## Идиома: бесконечный цикл

```ts
let rafId: number;

const tick = (now: number) => {
  const phase = (now % DURATION) / DURATION;     // 0..1 за период
  const value = compute(phase);
  applyToDom(value);                              // или setPaintProperty / canvas draw

  rafId = requestAnimationFrame(tick);            // запросить следующий кадр
};

rafId = requestAnimationFrame(tick);              // запустить

// cleanup:
cancelAnimationFrame(rafId);
```

Ключевая структура — **фаза вычисляется из `now`**, не из счётчика `i++`. Так анимация:
- не дрейфует, если кадр пропущен (фаза просто перепрыгнет вперёд);
- одинаково играет на 60 Hz и 120 Hz дисплеях;
- не «ускоряется» если внезапно в одном кадре два tick'а догнали друг друга.

Счётчик `i++` даёт «N кадров до конца» — это framerate-зависимо и ломается при дропе кадров.

## Гейт запуска

Не запускать RAF просто так. Цикл стоит дёшево, но **не нулевых** копеек: каждый кадр будит main thread, не даёт CPU уйти в idle, сжигает батарею на мобильном.

```ts
useEffect(() => {
  if (!hasAnimatedTarget) return;                 // нечего анимировать → не крутим

  let rafId = requestAnimationFrame(tick);
  return () => cancelAnimationFrame(rafId);       // важно: cleanup при unmount/смене deps
}, [hasAnimatedTarget]);
```

Типовые гейты: «есть ли элементы в активном состоянии», «находится ли пользователь на этом экране», `prefers-reduced-motion: reduce` → пропустить и применить статическое значение один раз.

## Где RAF — родной инструмент

- **Императивная анимация значений** которые нельзя выразить декларативным CSS: `setPaintProperty` на MapLibre/Mapbox, ручной draw на `<canvas>`, физика частиц, пружинки в Framer Motion.
- **Scroll-driven эффекты** на `scroll` event без jank: throttle через RAF (только последний scroll за кадр обрабатывается).
- **Двухфазный mount** в Presence-паттерне: смонтировать с `data-state="closed"`, через RAF переключить на `"open"` — браузер успевает зарегистрировать начальный стиль, CSS-transition срабатывает. См. [[presence-pattern]].
- **FLIP-анимации**: измерил → перерендерил → измерил → через RAF применил инвертирующий transform → следующий RAF убрал transform с transition.
- **Игровой loop**: state update → draw → следующий кадр.

## Где RAF — лишний

- **CSS transitions/keyframes** покрывают on/off и фиксированные траектории — там RAF не нужен, всё гонит compositor.
- **Долгие интервалы** (раз в секунду, минуту) — `setInterval` или `setTimeout` дешевле и понятнее. RAF про 16 мс, не про секунды.
- **Бэкграунд-работа** которая должна крутиться когда вкладка не активна (heartbeat, polling) — RAF поставится на паузу. Нужен `setInterval` или Web Worker.

## Подводные

- **Closure stale state**: внутри `tick` замыкается state на момент запуска эффекта. Если нужно читать актуальное — через `useRef`, не через прямой захват переменной.
- **Двойной запуск в React StrictMode**: эффект может смонтироваться дважды → два параллельных RAF-цикла. Обязательный `cancelAnimationFrame` в cleanup.
- **Pause через `document.visibilitychange`**: при возврате во вкладку первый `now` после паузы будет «прыжком». Если важна непрерывность — сохранять `lastNow` и считать фазу от прошедшего delta, а не от `now % period`.
- **120 Hz дисплеи** (ProMotion на Apple-устройствах, многие Android-флагманы): RAF позовётся **в два раза чаще**. Если внутри tick'а — тяжёлая работа, бюджет 8.3 мс, не 16.6 мс. Тяжёлые `setPaintProperty` или canvas draw нужно мерить именно там.

## Концепция шире

Паттерн «работа, синхронизированная с тактом потребителя» встречается всюду где есть фиксированный rendering pipeline или scheduler:

- **vsync / vblank в GPU** — игровой движок ждёт сигнал вертикального бланкинга монитора и рендерит ровно один кадр в окно. RAF — это vsync, прокинутый из GPU через браузер в JS.
- **Game loop в движках** — `update → render → wait for next frame`. Та же структура, тот же гейт «нечего обновлять — не крутим».
- **OS scheduler quantum** — задачи группируются в кванты времени, чтобы не дёргать CPU чаще чем нужно. Аналогично RAF группирует JS-работу к кадру.
- **React commit phase** — батчинг state-апдейтов до конца микротаска, один render вместо N. Тот же принцип «несколько причин для работы → одна выдача».
- **Database group commit** — несколько транзакций flush'аются на диск одним fsync. Привязка работы к такту нижележащего носителя.

Общее везде: **производитель работы (JS / задачи / транзакции) тактируется частотой потребителя (экран / CPU quantum / диск), а не своей собственной**. Это даёт два эффекта: (1) нет работы, которая всё равно невидима или незаметна (между кадрами/тактами), (2) разрозненные источники работы естественно сливаются в одну выдачу.
