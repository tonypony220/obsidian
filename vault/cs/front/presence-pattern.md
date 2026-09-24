---
title: Presence — удержание компонента в DOM во время анимации
tags: [concept, pattern, react, frontend]
date: 2026-04-21
---

# Presence

moc: [[front-moc]]
back: [[component-layers]]
next:
- [[portal-and-animation-separation]]
- [[on-exit-complete-callback]]
- [[compositor-layers]]
- [[request-animation-frame]]

---

```
time →

open:    [нет в DOM]  ─▶  [в DOM, state=closed]  ─▶  [в DOM, state=open]
                              первый кадр              CSS видит переход
                              ↑                        ↑
                           mount                    rAF через кадр

close:   [в DOM, state=open]  ─▶  [в DOM, state=closed]  ─▶  [нет в DOM]
                                    анимация 200ms             unmount
                                                               ↑
                                                           transitionend
```

**TL;DR:** `{isOpen && <Modal>}` размонтирует мгновенно — анимации закрытия физически не может быть. `<Modal isVisible>` всегда в DOM — ест ресурсы. Presence — обёртка, которая монтирует при появлении через промежуточный кадр `state=closed → open` и задерживает размонтирование до `transitionend`. Один раз написан, используется для Modal/Sheet/Toast/Tooltip.

## Что ломается без Presence

### Вариант А: всегда в DOM

```tsx
<Modal data-state={isOpen ? 'open' : 'closed'} />
```

- Модалка всегда смонтирована, даже закрытая.
- Focus trap, `Escape`-обработчики, portal'ы — активны всегда.
- Память, layout work, лишний DOM — особенно больно если таких элементов много (тосты, dropdown'ы).

### Вариант Б: условный рендер

```tsx
{isOpen && <Modal data-state="open" />}
```

- `isOpen=false` → React **мгновенно** убирает элемент. Нечему анимироваться.
- При открытии: элемент появляется сразу с `data-state="open"`. CSS нужен переход **от** closed **к** open — стартового кадра нет, анимации тоже.

## Что нужно для рабочей анимации

- **Mount**: нужен промежуточный кадр «в DOM, ещё closed», чтобы CSS зафиксировал стартовое значение → следующим кадром `open` → переход запускается.
- **Unmount**: не убирать из DOM мгновенно, а подождать `transitionend` (200мс) и только потом.

React сам этого не делает — `{cond && <X/>}` это синхронный mount/unmount без промежуточных фаз.

## Presence решает это

```tsx
<Presence present={isOpen}>
  <ModalCard data-state={isOpen ? 'open' : 'closed'}>...</ModalCard>
</Presence>
```

Поведение:
- `present: false → true` — монтирует ребёнка сначала с `data-state="closed"`, через `requestAnimationFrame` меняет на `"open"`. CSS видит переход → анимация.
- `present: true → false` — ставит `data-state="closed"`, подписывается на `transitionend`/`animationend` → после события размонтирует.

Ребёнок ничего не знает про lifecycle, только рендерит себя с атрибутом.

## Без Presence — вся логика внутри компонента

Если бы не было Presence, каждый анимируемый компонент тащил бы это сам:

```tsx
function Modal({ isOpen, children }) {
  const [mounted, setMounted] = useState(isOpen);
  const [state, setState] = useState<'open' | 'closed'>(isOpen ? 'open' : 'closed');
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (isOpen) {
      setMounted(true);
      requestAnimationFrame(() => setState('open'));
    } else {
      setState('closed');
      const el = ref.current;
      const onEnd = () => setMounted(false);
      el?.addEventListener('transitionend', onEnd);
      return () => el?.removeEventListener('transitionend', onEnd);
    }
  }, [isOpen]);

  if (!mounted) return null;
  return <div ref={ref} data-state={state}>{children}</div>;
}
```

Повторять это в Modal, Sheet, Toast, Tooltip, Dropdown — копипаста. Presence выносит этот танец в один primitive.

## CSS описывает анимацию, не JS

```css
.modal-card { transition: opacity 200ms, transform 200ms; }
.modal-card[data-state="open"]   { opacity: 1; transform: scale(1); }
.modal-card[data-state="closed"] { opacity: 0; transform: scale(0.95); }
```

`data-state` — декларативный контракт между Presence и CSS: JS просто переключает атрибут, браузер сам интерполирует.

## Готовые реализации

- **Radix `<Presence>`** — ~100 строк, чистый CSS-подход. Основа всех Radix-компонентов (Dialog, Popover, Tooltip). Лёгкий.
- **Framer Motion `<AnimatePresence>`** — мощнее: spring-физика, жесты, layoutId-анимации. ~50 KB. Подробно: [[animate-presence]].
- **react-transition-group** — старый низкоуровневый, API менее удобный.

По умолчанию — Radix Presence. Framer — когда нужны springs/gestures.

## Концепция шире

Паттерн «задержать завершение до конца асинхронного side-effect'а» — не только анимация:

- **React Suspense** — держит предыдущее UI, пока новое не загрузится; show/hide-переход не моментальный.
- **Graceful shutdown серверов** — не убивать процесс по SIGTERM, а дать закончить in-flight запросы.
- **Database connection draining** — перед закрытием пула ждать завершения транзакций.
- **Video/audio fade-out** — не обрывать звук мгновенно, микшировать до нуля.

Общее везде: **визуально/логически «закрыто» ≠ физически убрано**. Между этими двумя моментами — окно для завершения side-effect'а (анимация, запрос, звук).
