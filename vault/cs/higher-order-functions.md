---
title: Higher-Order Functions
tags: [concept, pattern]
date: 2026-04-12
---

# Функции высшего порядка

moc: [[fp-patterns-moc]]
next: [[pure-functions]] [[currying]] [[function-composition]]

---

```
принимает функцию:          возвращает функцию:
┌──────────────────┐        ┌──────────────────────┐
│ filter(fn)       │        │ withLogging(fn)       │
│    ├─ fn: x→bool │        │    └─► (...args) => { │
│    └─► [x,x,...] │        │         log(args)     │
└──────────────────┘        │         return fn()   │
                            │        }              │
                            └──────────────────────┘
```

Функция, которая принимает или возвращает другую функцию.

```ts
// принимает функцию
const adults = users.filter(u => u.age >= 18);

// возвращает функцию
function withLogging(fn: Function) {
    return (...args) => {
        console.log('calling with', args);
        return fn(...args);
    };
}

// и то и другое — middleware
const middleware = (next) => (action) => {
    console.log(action);
    return next(action);
};
```

## Где встречается

`map/filter/reduce`, Express middleware, React хуки (`useCallback`, `useMemo`), event handlers, декораторы.

## Зачем

Переиспользование логики без наследования. Вместо "класс наследует класс" — "функция оборачивает функцию".

## Происхождение

Lisp (1958) → JS `map/filter` (ES5, 2009).
