---
title: CSS specificity и class doubling
tags: [concept, css, frontend]
date: 2026-05-06
---

# CSS specificity

moc: [[front-moc]]
next:
- [[presence-pattern]]

---

```
 на одной кнопке <button class="button primary">

 селектор             specificity   что считается
 ─────────────────────────────────────────────────────────────────
 button                 0,0,1       1 element
 .primary               0,1,0       1 class
 button.primary         0,1,1       1 element + 1 class
 .button.primary        0,2,0       2 classes (chained = doubling)
 #hero button           1,0,1       1 ID + 1 element
 style="..."          1,0,0,0       inline (бьёт всё кроме !important)

 сравнение слева направо: первое отличие решает.
 0,2,0 > 0,1,1   потому что 2 > 1 в средней колонке.
```

**TL;DR:** когда несколько правил подходят на элемент, побеждает то, у которого **specificity** выше. Specificity — это кортеж `(inline, IDs, classes, elements)`, сравнивается слева направо. При равенстве — побеждает тот, что **позже** в каскаде. `.button.primary` (chained, два класса слитно) даёт `0,2,0` против `0,1,0` у `.primary` — это «doubling», практический приём чтобы перебить вероятные глобальные правила. Specificity — про **выбор победителя из нескольких правил**, не путать с inheritance — наследованием значений по дереву.

## Как считается specificity

Кортеж из четырёх чисел: `(a, b, c, d)`.

| Часть | Что считается |
|---|---|
| `a` | inline `style="..."` |
| `b` | количество ID-селекторов (`#hero`) |
| `c` | количество class-, attribute-, pseudo-class-селекторов (`.x`, `[type=text]`, `:hover`) |
| `d` | количество element-селекторов и pseudo-elements (`button`, `::before`) |

Часто пишут только `b,c,d` и опускают inline (если его нет).

Примеры:

```
button              → 0,0,0,1
button:hover        → 0,0,1,1   (:hover — pseudo-class)
.button             → 0,0,1,0
.button:hover       → 0,0,2,0
.button.primary     → 0,0,2,0   (два класса chained)
button.primary      → 0,0,1,1
#hero .button       → 0,1,1,0
```

## Как сравниваются

Слева направо, первое различие решает:

- `(0,1,0,0) > (0,0,9,9)` — один ID бьёт девять классов и девять элементов.
- `(0,0,2,0) > (0,0,1,1)` — два класса бьют один класс + один элемент.
- `(0,0,1,0) == (0,0,1,0)` — равенство, идём в каскад.

## Каскад при равном specificity

Побеждает правило, которое **позже** загружено/объявлено.

```css
/* file 1 */
.btn { color: red; }
/* file 2, импортирован после */
.btn { color: blue; }
/* → blue побеждает */
```

То же внутри одного файла — последняя строка побеждает.

## Inline и !important

```
.btn { color: red !important; }      /* специальный слой */
<div style="color: green">           /* inline = (1,0,0,0) */
```

Иерархия (сверху вниз = выше приоритет):

1. `!important` в авторских стилях.
2. Inline `style`.
3. Обычные правила, отсортированные по specificity.
4. User agent (browser default).

`!important` — escape hatch, не использовать без причины. Как только в проекте появляется два конкурирующих `!important`, всё ломается одинаково плохо.

## Class doubling как приём

Chained class selector `.a.b` (без пробела) — «элемент с обоими классами одновременно». Один элемент, но два class-сегмента в селекторе.

Не путать с descendant selector `.a .b` (с пробелом) — «`.b` внутри `.a`».

```html
<button class="button primary">
```

```css
.primary          { color: red; }   /* (0,0,1,0) */
.button.primary   { color: red; }   /* (0,0,2,0) — doubling */
```

Оба матчат одну и ту же кнопку — но второй имеет вдвое больший вес.

## Зачем doubling

Страховка от глобальных правил с element-селекторами:

```css
/* какой-то global.css */
button:hover { background: gray; }       /* (0,0,1,1) */

/* твой Button.module.css, bare */
.primary { background: red; }            /* (0,0,1,0) */
/* (0,0,1,1) > (0,0,1,0) — на hover'е красное проигрывает */

/* doubling */
.button.primary { background: red; }     /* (0,0,2,0) */
/* (0,0,2,0) > (0,0,1,1) — выигрывает */
```

Когда применять:
- В дизайн-системе: примитивы перебивают всё, что может оказаться в global.
- При миграции с глобальных стилей — пока global ещё жив.

Когда **не** применять:
- Global уже пустой и команда договорилась не возвращать element-стили — `.primary` достаточно.
- В application-level стилях, не в kit — там излишество.

## Specificity ≠ inheritance

Это два **разных** механизма каскада.

**Specificity** решает: «какое из применимых правил победит на этом элементе».

**Inheritance** решает: «значение какого свойства потечёт с предка на потомка, если у потомка не задано своё».

```css
button { font-weight: 600; }
```

- Specificity тут `(0,0,0,1)` — слабая.
- Но `font-weight` **наследуется**. Каждый `<span>`, `<div>`, `<p>` внутри `<button>` получит `600` через inheritance, **без какой-либо битвы specificity**.

Чтобы это перебить, потомку нужно явно `font-weight: 400`. Это не борьба selectors — это переопределение наследуемого значения.

Doubling против inheritance не помогает — у проблемы другая природа. Лечится удалением наследуемого свойства из global или явным reset на потомке.

## Концепция шире

«Кортеж приоритетов с tie-breaker по порядку» встречается везде, где есть конфликтующие правила:

- **DNS resolution** — `/etc/hosts` → `/etc/resolv.conf` → системный resolver, плюс TTL и порядок записей.
- **Routing tables** — longest prefix match (более специфичный маршрут побеждает), затем metric, затем порядок.
- **Linux iptables** — first match wins внутри chain, плюс приоритет chains.
- **Package manager dep resolution** — version constraints + lock-файл как tie-breaker.
- **Git merge strategy** — three-way merge с правилами разрешения конфликтов, fallback на manual resolve.

Общее везде: **при множественных применимых правилах нужен детерминированный порядок выбора — кортеж приоритетов сравнивается лексикографически, при равенстве работает заранее объявленный tie-breaker (порядок объявления, время, source)**. Без этого правила недетерминированы и баги невоспроизводимы.
