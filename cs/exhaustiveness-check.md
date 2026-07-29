---
title: Exhaustiveness Check
tags: [concept, type-system, pattern]
date: 2026-04-12
---

# Exhaustiveness Check

next:
- [[algebraic-data-types]]
- [[interface-vs-discriminated-union]]
- [[make-illegal-states-unrepresentable]]
- [[type-design-without-adt]]

---

```
type Shape = Circle | Rect | Triangle   ← добавили Triangle
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
          area.ts            describe.ts          render.ts
       ✗ not exhaustive    ✗ not exhaustive    ✗ not exhaustive
              └───────────────────┼───────────────────┘
                      компилятор нашёл все места
```

Компилятор проверяет, что все варианты discriminated union обработаны. Забыл case — ошибка компиляции, а не баг в рантайме.

## Как работает

Добавляем вариант в тип — одна строка:

```ts
type Shape =
    | { kind: 'circle'; radius: number }
    | { kind: 'rect'; width: number; height: number }
    | { kind: 'triangle'; base: number; height: number }  // +1 строка
```

Все функции с `switch` по `Shape` перестают компилироваться:

```
area.ts     — Function lacks ending return statement
describe.ts — Function lacks ending return statement
render.ts   — Function lacks ending return statement
```

Компилятор нашёл все три места. Чинишь по одному — добавляешь `case 'triangle'` в каждый switch. Не `grep` по кодовой базе, а компилятор как чеклист.

## До и после ADT

**Без ADT** — тип один, поля опциональные, компилятор ничего не проверяет:

```ts
type Shape = {
    kind: string;
    radius?: number;
    width?: number;
}

function area(s: Shape): number {
    if (s.kind === 'circle') {
        return Math.PI * s.radius! * s.radius!; // ! — "поверь мне"
    }
    return 0; // ← тихо вернём 0 для неизвестных
}
```

Проблемы: можно создать бессмыслицу `{ kind: 'circle', width: 10 }`, забытый вариант → тихий `return 0`, `!` повсюду.

**С ADT** — каждый вариант описан точно, компилятор проверяет всё:

```ts
type Shape =
    | { kind: 'circle'; radius: number }
    | { kind: 'rect'; width: number; height: number }

function area(s: Shape): number {
    switch (s.kind) {
        case 'circle': return Math.PI * s.radius * s.radius;
        case 'rect':   return s.width * s.height;
    }
}
```

Добавь вариант — функция перестаёт компилироваться. Не тихий `return 0`, а ошибка на этапе сборки.

## Языки без exhaustiveness check

Не все языки поддерживают закрытые наборы вариантов:

**Java (до sealed classes)** — `instanceof` без гарантий:
```java
double area(Shape s) {
    if (s instanceof Circle c) return Math.PI * c.radius * c.radius;
    throw new IllegalArgumentException(); // рантайм, не компиляция
}
```

**Go** — type switch без проверки полноты:
```go
switch v := s.(type) {
case Circle: return math.Pi * v.Radius * v.Radius
}
return 0 // тихий ноль
```

**C** — union + enum вручную, никаких проверок. Забыл проверить kind → мусор в памяти.

Общий паттерн: без [[algebraic-data-types]] языки эмулируют "одно из" через наследование или ручные теги, но exhaustiveness check теряется.
