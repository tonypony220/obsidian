---
title: Slot architecture — компонент как набор именованных слотов
tags: [concept, pattern, react, frontend]
date: 2026-04-23
---

# Slot architecture

moc: [[front-moc]]
back: [[component-layers]]
next:
- [[radix-primitives]]
- [[react-context]]

---

```
  Монолитный API                        Slot-архитектура

  ┌────────────────────────┐            ┌──────────────────────────┐
  │ <Dialog                │            │ <Dialog.Root>            │
  │   open={...}           │            │   <Dialog.Trigger/>      │
  │   title="..."          │            │   <Dialog.Portal>        │
  │   description="..."    │            │     <Dialog.Overlay/>    │
  │   trigger={<Button/>}  │            │     <Dialog.Content>     │
  │   overlay={<Custom/>}  │            │       <Dialog.Title/>    │
  │   closeButton={<X/>}   │            │       <Dialog.Description/>
  │   showOverlay={true}   │            │       <Dialog.Close/>    │
  │   portalContainer={…}  │            │     </Dialog.Content>    │
  │   ... ×20 props />     │            │   </Dialog.Portal>       │
  └────────────────────────┘            │ </Dialog.Root>           │
   один компонент, гора пропсов          └──────────────────────────┘
                                          набор подкомпонентов,
                                          общающихся через Context
```

**TL;DR:** Slot-архитектура — разбиение одного UI-компонента на множество именованных подкомпонентов (`Dialog.Root`/`Trigger`/`Content`/`Close`), собираемых пользователем в композицию. Альтернатива монолиту с десятками пропсов. Подкомпоненты общаются через React Context. Это современная реализация **compound components pattern** (Kent C. Dodds, 2017).

## Как было бы без этого

Монолит:

```tsx
<Dialog
  open={isOpen}
  onClose={close}
  title="Delete item?"
  description="This is permanent"
  trigger={<Button>Open</Button>}
  overlay={<CustomOverlay/>}
  closeButton={<X/>}
  showCloseButton={true}
  portalContainer={document.body}
  closeOnEsc={true}
  closeOnOutsideClick={true}
  preventScrollLock={false}
/>
```

Что ломается:

1. **Пропсов всегда не хватает.** Захотел два разных Close-элемента (один в header, второй в footer) → нужен новый prop. Захотел добавить кастомный header сверху Title → ещё prop (`headerSlot={...}` — но это уже фактически слот, просто кривой). Через год у компонента 30+ пропсов.

2. **Кастомизация = форк.** Хочешь обернуть Content в свой scroll-контейнер? Нельзя — структура зашита внутри. Форкаешь компонент целиком.

3. **Render-props спираль.** Чтобы хоть как-то дать гибкость, делают render-функции:
   ```tsx
   <Dialog
     renderHeader={({ close }) => <MyHeader onClose={close}/>}
     renderFooter={({ confirm, cancel }) => <Buttons.../>}
     renderBody={({ data }) => <Custom.../>}
   />
   ```
   Это **уже слоты, просто в виде функций** — но менее читаемо, чем JSX-композиция, и хуже для типизации.

4. **Структура не видна.** По вызову `<Dialog title=... description=...>` нельзя понять, в каком порядке всё рендерится, есть ли overlay, где закрывающая кнопка. Надо лезть в исходник.

## Slot-архитектура

```tsx
<Dialog.Root open={isOpen} onOpenChange={setOpen}>
  <Dialog.Trigger asChild>
    <Button>Open</Button>
  </Dialog.Trigger>

  <Dialog.Portal>
    <Dialog.Overlay />
    <Dialog.Content>
      <Dialog.Title>Delete item?</Dialog.Title>
      <Dialog.Description>This is permanent</Dialog.Description>

      <Button onClick={confirm}>Confirm</Button>
      <Dialog.Close>Cancel</Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

Каждый подкомпонент = именованная роль в архитектуре. Структура видна в JSX.

## Как работает внутри: Context

Подкомпоненты не получают пропсы друг от друга — общаются через **React Context**:

```tsx
const DialogContext = createContext<{
  open: boolean;
  setOpen: (v: boolean) => void;
}>(null!);

function Root({ open, onOpenChange, children }) {
  return (
    <DialogContext.Provider value={{ open, setOpen: onOpenChange }}>
      {children}
    </DialogContext.Provider>
  );
}

function Trigger({ children }) {
  const { setOpen } = useContext(DialogContext);
  return <button onClick={() => setOpen(true)}>{children}</button>;
}

function Content({ children }) {
  const { open } = useContext(DialogContext);
  return (
    <Presence present={open}>
      <div role="dialog" data-state={open ? 'open' : 'closed'}>
        {children}
      </div>
    </Presence>
  );
}

function Close({ children }) {
  const { setOpen } = useContext(DialogContext);
  return <button onClick={() => setOpen(false)}>{children}</button>;
}

export const Dialog = { Root, Trigger, Portal, Overlay, Content, Title, Close };
```

Ключ: `Root` провайдит state в Context, дочерние слоты читают. **Пользователю не нужно пробрасывать пропсы между Trigger, Content и Close** — они находят друг друга через контекст.

## Compound Components Pattern

Slot-архитектура у Radix — это современная упаковка паттерна **compound components**, описанного Kent C. Dodds в 2017. Базовый пример того времени:

```tsx
<Toggle onToggle={toggle}>
  <Toggle.On>Value is on</Toggle.On>
  <Toggle.Off>Value is off</Toggle.Off>
  <Toggle.Button />
</Toggle>
```

Тогда же это часто реализовывали через `React.Children.map` + `cloneElement` — родитель пробегал по детям и инжектил пропсы. Сейчас все ушли на **Context** — масштабируется лучше (работает через произвольную глубину вложенности).

Radix довёл паттерн до продакшена:
- Доступность из коробки (focus trap, ARIA, Escape, click outside).
- Controlled/uncontrolled через `open` vs `defaultOpen` ([[controlled-vs-uncontrolled]]).
- Анимации через `<Presence>` ([[presence-pattern]]).
- Полиморфизм через `asChild` ([[as-vs-aschild]]).

## Связь с Web Components `<slot>`

В HTML-стандарте Web Components есть нативный `<slot>`:

```html
<template id="my-card">
  <div class="card">
    <slot name="header"></slot>
    <slot name="body"></slot>
  </div>
</template>

<my-card>
  <h1 slot="header">Title</h1>
  <p slot="body">Text</p>
</my-card>
```

Тот же принцип: компонент объявляет именованные точки вставки, пользователь заполняет. В React нативного `<slot>` нет, но конвенция `Component.PartName` — функциональный аналог.

## Цена

- **Больше JSX** — структура на виду, для новичка может выглядеть громоздко.
- **Легко собрать неправильно**: забыть `Portal`, неправильно вложить `Close` — runtime-ошибка или просто не работает.
- **Сложнее типизация**: связь между подкомпонентами через Context — TS не всегда ловит «ты использовал `Dialog.Trigger` вне `Dialog.Root`».
- **Boilerplate простых случаев**: для тривиального диалога 5 строк JSX вместо одного.

## Когда применять

- Компонент имеет **переменную структуру**: разное число элементов, опциональные части, кастомизируемое содержимое (Dialog, Menu, Accordion, Tabs, Form).
- API на 5+ пропсах, и явно будет расти.
- Несколько детей одного типа (`Tab` × N, `MenuItem` × N).

**Не нужно**, если:
- Компонент атомарный с фиксированной структурой (`Button`, `Input`, `Avatar`).
- Структура не варьируется и не будет.

## Концепция шире

Та же идея «декомпозиция через объявленные роли вместо одного интерфейса» встречается везде:

- **DI-контейнеры** (Spring, NestJS): объявляешь `@Component`/`@Inject`-точки, контейнер сам собирает граф. Альтернатива — гигантский конструктор со всеми зависимостями.
- **Plugin-системы** (Webpack, Vite, ESLint): hooks/рекомендуемые точки расширения вместо одного файла-конфига на тысячу опций.
- **Unix pipelines**: маленькие утилиты в pipeline вместо одного `swiss-army-knife`.
- **Vue scoped slots / Svelte slots**: тот же паттерн как нативная фича фреймворка.

Общее везде: **гибкость через композицию vs гибкость через конфигурацию**. Конфиг растёт линейно с требованиями (каждая новая опция = новый prop); композиция растёт логарифмически (новые требования покрываются комбинациями существующих частей).
