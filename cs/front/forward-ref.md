---
title: forwardRef — прокидывание ref через компонент
tags: [concept, react, frontend]
date: 2026-04-21
---

# forwardRef

moc: [[front-moc]]
back: [[component-layers]]
next:
- [[use-imperative-handle]]
- [[polymorphic-as-prop]]

---

```
  родитель                 компонент            реальный DOM
 ┌──────────┐              ┌──────────┐         ┌──────────┐
 │ useRef() │─── ref ─────▶│forwardRef│── ref ─▶│  <div>   │
 │          │              │          │         │          │
 │ r.current│◀─── React записывает сюда ───────│ HTMLDiv  │
 └──────────┘              └──────────┘         └──────────┘
```

**TL;DR:** `ref` работает автоматически только на DOM-узлах; на пользовательских компонентах React его игнорирует. `forwardRef` — обёртка, которая принимает ref вторым аргументом, чтобы компонент мог привязать его к своему внутреннему DOM.

## Зачем нужен ref вообще

Родителю иногда нужна ссылка на **настоящий DOM-узел** внутри компонента:

- `element.focus()` — сфокусировать инпут после mount
- `element.scrollIntoView()` — прокрутить к сообщению в чате
- `element.getBoundingClientRect()` — измерить размер
- отдать DOM внешней библиотеке (drag, popper, animation)

## Как ref работает на чистом DOM

```tsx
const inputRef = useRef<HTMLInputElement>(null);

useEffect(() => {
  inputRef.current?.focus();
}, []);

return <input ref={inputRef} />;
```

Шаги:
1. `useRef(null)` создаёт объект `{ current: null }`.
2. React рендерит `<input>`, вставляет в DOM.
3. React видит `ref={inputRef}` на **реальном DOM-элементе** и сам записывает: `inputRef.current = <input>`.
4. `useEffect` после mount → `.focus()` работает.

Ключ: `ref` — не обычный prop. React ловит его на JSX-уровне и сам кладёт DOM в `.current`.

## Что ломается с компонентом

```tsx
function MyInput(props) {
  return <input {...props} />;
}

const r = useRef<HTMLInputElement>(null);
<MyInput ref={r} />   // ❌ r.current === null
```

React видит `ref` на компоненте (функции), не знает, к какому из внутренних DOM-узлов его прицепить — и **игнорирует**. До функции `MyInput` `ref` не доходит, потому что это не обычный prop.

## Решение — forwardRef

```tsx
const MyInput = forwardRef<HTMLInputElement, InputProps>((props, ref) => (
  <input ref={ref} {...props} />
));

<MyInput ref={r} />   // ✅ r.current = <input>
```

`forwardRef` говорит React: «этот компонент умеет принимать ref, передай мне его вторым аргументом». Компонент сам решает, к какому внутреннему DOM-узлу его привязать.

## Компонент не владеет ref

`ref` — это объект родителя. Компонент только **прокидывает** его к своему DOM. Сам с ним работать не должен — это полезет в ответственность родителя.

Если примитиву самому нужен DOM-узел — заводит **свой внутренний** ref:

```tsx
const Box = forwardRef<HTMLDivElement, Props>((props, ref) => {
  const internal = useRef<HTMLDivElement>(null);
  // internal — для Box'а, ref — для родителя
  return <div ref={ref} {...props} />;
});
```

Нужен и наружу, и внутрь — `mergeRefs` или [[use-imperative-handle]].

## Почему в primitives — всегда

Primitive переиспользуется в десятках мест. Ты не знаешь заранее, кто из контейнеров захочет сфокусить, измерить, скролльнуть. Добавить `forwardRef` — одна строка. Не добавить → через месяц придётся переписывать 40 вызовов.

## React 19

В React 19 `forwardRef` не нужен — `ref` стал обычным prop:

```tsx
function Box({ ref, ...props }: BoxProps & { ref?: Ref<HTMLDivElement> }) {
  return <div ref={ref} {...props} />;
}
```

При апгрейде с React 18 просто снимается обёртка.

## Концепция шире

Та же идея «протянуть handle через слой абстракции, не ломая инкапсуляцию» встречается везде:

- **C/POSIX**: `fopen()` возвращает `FILE*` — opaque handle, владелец может читать/писать, но не знает как устроен файловый дескриптор внутри.
- **ООП**: геттер возвращает ссылку на внутренний объект — клиент управляет, но объект остаётся членом класса.
- **gRPC streams**: сервер отдаёт клиенту handle стрима; клиент пушит данные, но не владеет транспортом.
- **OpenGL**: `glGenTextures()` даёт числовой id — клиент использует, GPU владеет памятью.

Общее везде: **владелец ресурса и пользователь — разные сущности**; между ними прокидывается handle, через который пользователь дёргает методы ресурса, не зная его устройства.
