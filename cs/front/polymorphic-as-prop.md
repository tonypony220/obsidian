---
title: Polymorphic as prop — смена тега без дублирования компонентов
tags: [concept, pattern, react, frontend]
date: 2026-04-21
---

# Polymorphic `as` prop

moc: [[front-moc]]
back: [[component-layers]]
next:
- [[as-vs-aschild]]
- [[controlled-vs-uncontrolled]]

---

```
  <Box as="section" p="lg">       <Box as="nav" p="md">        <Stack as="ul" gap="sm">
         │                               │                              │
         ▼                               ▼                              ▼
  <section class=".." style=".."> <nav class=".." style="..">   <ul class=".." style="..">

   ── один компонент, любой semantic HTML ──
```

**TL;DR:** prop `as` позволяет одному primitive рендерить любой HTML-тег, сохраняя стили и API. Альтернатива — `asChild` / Slot (Radix): композиция вместо замены тега; сложнее, но нужно для композиции с готовыми компонентами (Link из роутера и т. п.).

## Проблема

HTML-теги несут семантику, не только визуал. Одна и та же карточка-контейнер может быть:

- `<section>` — раздел
- `<nav>` — навигация
- `<article>` — самостоятельный блок
- `<h2>` — заголовок
- `<ul>` — список

Семантика важна для:
- **Screen readers** — слепой прыгает по `<nav>` и `<h2>` клавишей, `<div>` игнорирует.
- **SEO** — поисковик понимает структуру страницы.
- **Дефолтное поведение браузера** — `<button>` сабмитит форму, `<a>` открывается в новой вкладке.

Плодить `BoxDiv`, `BoxSection`, `BoxNav`, `BoxH2` — дублирование.

## Решение: prop `as`

```tsx
<Box as="section" p="lg">...</Box>    // <section class="box" style="padding:16">
<Box as="nav" p="md">...</Box>        // <nav ...>
<Stack as="ul" gap="sm">...</Stack>   // <ul ...>
```

Один компонент, любая семантика, те же токены и API.

## Реализация

```tsx
type BoxProps = {
  as?: ElementType;        // строка-тег или компонент
  p?: SpacingToken;
  bg?: ColorToken;
  children?: ReactNode;
};

const Box = ({ as: Component = 'div', p, bg, ...rest }: BoxProps) => (
  <Component
    style={{ padding: spacing[p!], background: colors[bg!] }}
    {...rest}
  />
);
```

Ключевые моменты:
1. **Переименование** `as` → `Component` (с большой буквы). JSX требует заглавную букву: `<Component>` — переменная-компонент, `<as>` — литеральный HTML-тег `<as>`.
2. `ElementType` — покрывает и строковые теги, и React-компоненты.
3. Дефолт `'div'` — если `as` не передали.

## Типизация

Базовая версия теряет связь пропсов с тегом:

```tsx
<Box as="a" href="/home" />    // href валиден для <a>, но TS это не проверит
<Box as="div" href="/home" />  // бессмысленно, но пройдёт
```

Полноценная версия — через generic:

```tsx
type BoxProps<T extends ElementType> = {
  as?: T;
  /* свои пропсы */
} & ComponentPropsWithoutRef<T>;

function Box<T extends ElementType = 'div'>({ as, ...props }: BoxProps<T>) {
  const Component = as ?? 'div';
  return <Component {...props} />;
}
```

Edge cases с forwardRef + generic нетривиальны (отдельная тема). Простая версия без generic — рабочий компромисс.

## Альтернатива: `asChild` (Radix-style)

Вместо замены тега — композиция через слот:

```tsx
<Button asChild>
  <Link to="/home">Go home</Link>
</Button>
// => <a class="button" href="/home">Go home</a>
```

Button не рендерит свой тег, а **навешивает стили и поведение на child**.

Плюсы: типизация проще (child сам себя типизирует), работает с кастомными компонентами (Link из роутера).
Минусы: требует Slot-утилиту (мердж refs, className, style, event handlers) — самому писать дорого; ровно один child обязателен.

Подробно про tradeoff: [[as-vs-aschild]].

## Когда НЕ нужно

- Компонент всегда конкретный тег: `<Input>` всегда `<input>`, `<Canvas>` всегда `<canvas>`.
- Теги с радикально разным API (`<video>` vs `<div>`) — проще два отдельных компонента.

## Концепция шире

Паттерн «отделить что делать от чем делать» — переменная реализация инжектится снаружи:

- **Strategy pattern** (OOP): `Sorter(comparator)` — алгоритм подставляется снаружи.
- **HOC / higher-order functions**: `withAuth(Component)` — оборачиваем любую функцию.
- **Unix**: `CC=clang make`, `PAGER=bat cmd` — подставляемая реализация через переменную окружения.
- **DI-контейнеры**: интерфейс декларирован, реализация резолвится в runtime.

Общее везде: **интерфейс фиксирован, реализация варьируется** — полиморфизм как средство переиспользования без дублирования.
