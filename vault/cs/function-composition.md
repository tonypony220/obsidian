---
title: Function Composition
tags: [concept, pattern]
date: 2026-04-12
---

# Композиция функций

moc: [[fp-patterns-moc]]
next: [[higher-order-functions]] [[pure-functions]]

---

```
input ──► validate ──► normalize ──► save ──► output
           f(x)          g(x)        h(x)
                  h(g(f(input)))
```

Собирать сложное из простых функций. Вместо наследования — цепочка трансформаций.

```ts
// pipe — данные текут через функции слева направо
const process = pipe(
    validate,
    normalize,
    save,
);
process(input); // validate → normalize → save

// в реальности чаще просто цепочки
const result = items
    .filter(isActive)
    .map(toDTO)
    .sort(byDate);
```

## Зачем

Каждая функция делает одно дело, легко переставлять/заменять шаги. `a.filter().map().sort()` читается как конвейер.

## Происхождение

Математика (композиция функций) → Unix pipes → lodash/fp.
