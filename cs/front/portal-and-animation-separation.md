---
title: Portal и animation — три ответственности, три примитива
tags: [concept, pattern, react, frontend]
date: 2026-04-22
---

# Portal и animation — раздельно

moc: [[front-moc]]
back: [[component-layers]]
next:
- [[slot-architecture]]
- [[radix-primitives]]

---

```
 <Presence present={isOpen}>       ← жизненный цикл (mount/unmount с анимацией)
   <Portal>                        ← перенос в document.body
     <Sheet data-state="open">     ← разметка и стили
       {children}
     </Sheet>
   </Portal>
 </Presence>
```

**TL;DR:** Portal (перенос в body), Presence (mount/unmount с анимацией), Sheet/Modal/Toast (разметка) — три независимых primitive'а. Смешивать в один компонент означает копипасту при каждом новом overlay-элементе и блокирует переиспользование.

## Что такое Portal

`ReactDOM.createPortal(children, domNode)` — рендерит children физически **в другом месте DOM**, оставляя их **логически потомком** в React-дереве (контексты работают, события всплывают).

Зачем модалкам:

```
JSX:                          DOM без портала:        DOM с порталом:
<App>                         <body>                   <body>
  <Card overflow:hidden>        <card                    <card/>                ← тут пусто
    <Modal/>                      overflow:hidden>       <modal>✅ на top</modal>
  </Card>                         <modal>❌ обрезана   </body>
</App>                        </body>
```

Без портала модалка обрезается родительским `overflow: hidden`, ограничивается z-index стэкинг-контекстом, искажается transform-родителя. Portal её вытаскивает в `document.body`, где она свободна.

## Что ломается, когда всё в одном компоненте

```tsx
function CenterSheet({ isOpen, children }) {
  if (!isOpen) return null;
  return createPortal(<div className="sheet">{children}</div>, document.body);
}
```

Три ответственности смешаны:
1. Жизненный цикл (`if !isOpen return null`).
2. Перенос в body (`createPortal`).
3. Layout/стили (`<div className="sheet">`).

Последствия:
- Захочешь анимацию — `return null` мгновенно убивает элемент, надо втаскивать Presence-логику сюда же.
- Нужен inline-drawer без портала — придётся форкать.
- Тост с другими стилями — снова форк Portal-обёртки.
- Тестировать Portal в отрыве от Sheet нельзя.

## Разделение на три primitive'а

```tsx
<Presence present={isOpen}>
  <Portal>
    <Sheet data-state={isOpen ? 'open' : 'closed'}>
      {children}
    </Sheet>
  </Portal>
</Presence>
```

| Primitive | Ответственность | Размер |
|---|---|---|
| `Presence` | mount/unmount с анимацией ([[presence-pattern]]) | ~100 строк |
| `Portal` | перенос в `document.body` | ~5 строк |
| `Sheet` | разметка и стили | по задаче |

Portal сам по себе — почти ничего:

```tsx
function Portal({ children, container }: Props) {
  return createPortal(children, container ?? document.body);
}
```

## Выгоды

- **Переиспользование**: Portal — для Modal, Sheet, Toast, Tooltip, Popover. Presence — для всего, что анимируется. Sheet — только там, где нужен sheet-layout.
- **Свободные комбинации**: inline-drawer (`<Presence><Sheet/></Presence>`), тост без sheet-стилей (`<Presence><Portal><Toast/></Portal></Presence>`), модалка с другой анимацией (тот же Presence, другой layout).
- **Независимые тесты**: каждый primitive тестируется сам по себе.

## Порядок обёрток

Канонический:

```
<Presence>    ← решает, жив ли ребёнок сейчас
  <Portal>    ← если жив, переносит в body
    <Sheet>   ← разметка
```

Если поменять на `<Portal><Presence><Sheet/></Presence></Portal>` — Portal держит контейнер в `body` всегда, внутри него появляется/исчезает Sheet. Работает, но семантически странно: пустой invisible-узел живёт в `body` бесконечно.

## Откуда это

- **`createPortal`** — React core API (с 2017, React 16).
- **`Presence`** — популяризирован Radix, до него были `react-transition-group`, `react-motion`.
- **Декомпозиция Root/Trigger/Portal/Overlay/Content** — Radix-конвенция. Скопирована shadcn/ui, Headless UI и др.
- **`data-state="open"/"closed"`** — Radix-изобретение, стало стандартом.
- **Сам принцип «три ответственности — три примитива»** — общий React-подход (composition over configuration).

## Концепция шире

Паттерн «одна ответственность на модуль» встречается везде, где есть композиция:

- **Unix**: pipeline из мелких утилит (`grep | sort | uniq`), каждая делает одно.
- **HTTP middleware**: auth, logging, compression, routing — отдельные слои в цепочке.
- **Linux networking**: iptables, routing table, netfilter — разные подсистемы, объединяются поверх netdev.
- **Docker**: namespaces + cgroups + union-fs — три независимых механизма, вместе = контейнер.

Общее везде: **малые блоки с узкой ответственностью компонуются свободнее, чем один большой**. Цена композиции — немного синтаксического шума, выгода — экспоненциальная в переиспользовании.
