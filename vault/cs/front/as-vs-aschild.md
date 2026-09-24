---
title: as vs asChild — два способа сделать компонент полиморфным
tags: [tradeoff, react, frontend]
date: 2026-04-21
---

# `as` vs `asChild`

moc: [[front-moc]]
back: [[polymorphic-as-prop]]

---

```
as:                                    asChild:
<Box as="section">                     <Button asChild>
   ───▶ <section class="box">             <Link to="/">click</Link>
                                       </Button>
                                          ───▶ <a class="button" href="/">click</a>

  смена ТЕГА                            композиция с ГОТОВЫМ компонентом
```

**TL;DR:** `as` проще и дешевле, но работает только со сменой тега; `asChild` сложнее внутри (нужен Slot: мердж refs/className/events), но позволяет композить с готовыми компонентами вроде `<Link>` роутера.

## Таблица сравнения

| Аспект | `as` | `asChild` / Slot |
|---|---|---|
| API | `<Box as="section">` | `<Button asChild><Link/></Button>` |
| Что делает | заменяет тег | навешивает стили/поведение на child |
| Реализация | 5 строк (`<Component {...props}>`) | сотни строк (Slot: merge refs/className/events/style) |
| Типизация | generic (сложно для edge cases) | child сам себя типизирует |
| Работа с кастомным Link | ломается (Link хочет `to`, не `href`) | работает естественно |
| Требования к children | любой состав | ровно 1 элемент, иначе runtime error |
| Cognitive cost | низкий | средний (понять Slot) |

## Что умеет Slot внутри

Когда пишешь `<Button asChild><Link/></Button>`, Slot должен:

1. **Merge refs** — ref от Button и от Link оба привязать к DOM (`composeRefs`).
2. **Merge className** — `button__base` (от Button) + собственный класс Link.
3. **Chain event handlers** — `onClick` от Button и `onClick` от Link оба отработать по порядку.
4. **Merge style** — inline-стили Button поверх стилей Link с правильным приоритетом.
5. **Проверить cardinality** — ровно один child, иначе `React.Children.only` падает.

Самому делать — легко пропустить edge case. В Radix это библиотечный `Slot` (~150 строк).

## Когда что использовать

**`as`** — смена тега без composition:
- `<Box as="section">`, `<Stack as="ul">`, `<Text as="h2">`.
- 90% кейсов примитивов в дизайн-системе.
- Можно писать самому без библиотек.

**`asChild`** — композиция с готовым компонентом, у которого своя логика:
- Роутерный Link (`<Button asChild><Link to="/">...</Link></Button>`).
- Сторонние Trigger-компоненты.
- Когда child решает рендер тега сам (Link может рендерить `<a>` или собственный элемент).

## Смешивать не стоит

Иметь в одном компоненте и `as`, и `asChild` — путаница для пользователя API. Выбери один подход на слой.

## Почему Radix выбрал `asChild`

У Radix компоненты — не примитивы уровня Box, а **поведенческие** (Dialog.Trigger, Tooltip.Trigger, DropdownMenu.Item). Их часто композят с другими компонентами (кастомные кнопки, Link-и). Замена тега через `as` тут не хватает — композить нужно полноценные React-компоненты, не строки.

## Практический совет

Стартуй с `as`. Когда упрёшься в «нужно композить с `<Link>` из роутера» — тяни Radix Slot или Radix целиком. Не пиши свой Slot с нуля — дорого и легко сломать.
