---
title: onExitComplete — контракт «анимация закрытия завершена»
tags: [pattern, react, animation, frontend]
date: 2026-04-23
---

# onExitComplete

moc: [[front-moc]]
back: [[presence-pattern]]
next:
- [[pending-action-ref-latch]]
- [[fsm-closing-state]]

---

```
caller                         Presence                       browser
  │                               │                              │
  │ setOpen(false) ───────────▶  data-state="closed"             │
  │                               │                              │
  │                               │ addEventListener             │
  │                               │  ('animationend')            │
  │                               │                              │
  │                               │                   CSS keyframes → …
  │                               │                              │
  │                               │  ◀──────────── animationend ─│
  │                               │                              │
  │                               │ setMounted(false)            │
  │                               │ onExitComplete()             │
  │  ◀─── callback ───────────────│                              │
  │                                                              │
  │ dispatch SELECT_PROFILE       ← только теперь                │
  ▼                                                              │
```

**TL;DR:** exit-анимация растянута во времени: между «попросил закрыться» и «реально закрылось» — ~150-200 мс. Если в этот промежуток начать следующее действие (открыть новое overlay, пересобрать DOM), race ломает exit-анимацию. Решение — не угадывать таймером, а подписаться на `animationend` внутри Presence и экспортировать это наверх через `onExitComplete`-callback. Caller запоминает намерение через [[pending-action-ref-latch]], исполняет его из `onExitComplete`.

## Почему нельзя dispatch'ить сразу после setOpen(false)

Синхронный мир:

```tsx
const onClick = () => { onSelect(profile) };
```

С анимацией:

```tsx
const onClick = () => {
  setOpen(false);        // начал exit-анимацию (~150ms)
  onSelect(profile);     // СРАЗУ же тригерит новый коммит
};
```

Что происходит в браузере в тот же кадр:

1. Presence поставил `data-state="closed"` на уходящем элементе, анимация стартовала.
2. `onSelect` → reducer → монтируется новый большой overlay в `document.body`.
3. Style recalc на всём `<body>`, пересоздание stacking-context'ов, Paint.
4. Main thread занят → см. [[compositor-layers]]:
   - если уходящий элемент не был pre-promoted — анимация скипается
   - если stacking-context предка меняется — анимация обрывается без `animationend`
   - если key выше по дереву меняется — узел remount'ится, listener на `animationend` висит на detached-ноде

Итог: `animationend` не приходит, `setMounted(false)` не вызывается, панелька зависает в DOM с `data-state="closed"` навсегда.

## Плохие решения

**`setTimeout`:**

```tsx
setOpen(false);
setTimeout(() => onSelect(profile), 200);
```

- На медленной машине 200 мс мало → снова race.
- На быстрой — лишняя задержка отклика.
- `prefers-reduced-motion` → 200 мс пустоты.
- Число магическое, живёт вдалеке от CSS, который его породил.

**`requestAnimationFrame`:** сдвигает dispatch на один кадр (~16 мс). Race просто сдвигается на 16 мс правее.

**Snapshot props в ref:** компонент начинает помнить «что было до закрытия», рендерит из ref'а. Параллельный FSM внутри leaf'а. Ломает разделение ответственности — логика состояния протекает в view.

## Правильный контракт: animationend как событие

Браузер сам знает, когда анимация закончилась, и сообщает через `animationend`. Presence подписывается на него:

```tsx
function Presence({ present, children, onExitComplete }) {
  const [mounted, setMounted] = useState(present);

  useLayoutEffect(() => {
    if (present) setMounted(true);
  }, [present]);

  useLayoutEffect(() => {
    if (present || !mounted) return;
    const node = /* ref на DOM ребёнка */;
    const onEnd = () => {
      setMounted(false);
      onExitComplete?.();
    };
    node.addEventListener('animationend', onEnd);
    return () => node.removeEventListener('animationend', onEnd);
  }, [present, mounted]);

  return mounted
    ? cloneElement(children, { 'data-state': present ? 'open' : 'closed' })
    : null;
}
```

Три момента:
1. `present: false` → `data-state="closed"`, анимация стартует.
2. Жду `animationend`.
3. Размонтирую + зову `onExitComplete`.

## Inversion of control

Было: caller говорит «закройся, **я уже делаю следующее**».
Стало: caller говорит «закройся, **позови меня когда будешь готов**».

Presence становится **агентом времени** — владеет знанием «анимация закончилась», экспортирует его наверх через callback. Caller больше не гадает про тайминг.

## Типовое применение

```tsx
const pendingRef = useRef<Profile | null>(null);

const onClick = (profile) => {
  pendingRef.current = profile;   // запомнил намерение
  setOpen(false);                 // начал закрытие
};

<CenterSheet
  isOpen={open}
  onExitComplete={() => {
    const p = pendingRef.current;
    pendingRef.current = null;
    if (p) onSelect(p);           // исполнил после exit'а
  }}
>
```

Тайминг:

```
t=0       onClick:      ref = profile; isOpen = false
t≈0       Presence:     data-state="closed", анимация стартует
t≈150ms   browser:      animationend fires
t≈150ms   Presence:     setMounted(false) → DOM отцепляется
t≈150ms   Presence:     onExitComplete() → dispatch SELECT_PROFILE
t≈150ms   React commit: монтируется новый overlay
```

Никакой параллельности — каждый шаг начинается только когда предыдущий **реально** завершён.

Ref нужен потому, что между кликом и `onExitComplete` не должно быть ре-рендеров — см. [[pending-action-ref-latch]].

## Когда мало одного onExitComplete

Если данные для exit-view живут **не в caller'е**, а в самом FSM — caller не может просто запомнить намерение в ref и дождаться callback'а. Нужно расширить FSM отдельным состоянием `closing`, в котором данные ещё живы. См. [[fsm-closing-state]].

## Концепция шире

Паттерн «не гадай таймером, подпишись на событие окончания» — общий принцип асинхронного программирования:

- **`child_process.exec` vs `setTimeout(checkDone, 1000)`** — ждать `close` event, не опрашивать.
- **HTTP long polling → SSE/WebSocket** — не пинговать сервер, сервер сам шлёт push когда готов.
- **Database CDC (change data capture)** vs polling таблицы — подписка на binlog вместо `SELECT … WHERE updated_at > …`.
- **`fs.watch` vs `setInterval(fs.stat)`** — inotify/FSEvents вместо опроса.
- **Promise / async-await** — язык выражает «продолжи когда резолвится», без таймеров.

Общее везде: **источник события сам знает когда оно произошло — дай ему канал чтобы сообщить, не угадывай снаружи**. Inversion of control переносит знание туда, где оно живёт.
