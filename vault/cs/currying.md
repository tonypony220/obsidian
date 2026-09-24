---
title: Currying
tags: [concept, pattern]
date: 2026-04-12
---

# Каррирование / частичное применение

moc: [[fp-patterns-moc]]
next: [[higher-order-functions]] [[function-composition]]

---

```
f(a, b)  →  f(a)(b)          partial application
─────────────────────────────────────────────
multiply(2, 5) = 10
multiply(2)    → double       double(5) = 10
multiply(3)    → triple       triple(5) = 15
  ↑ зафиксировали a             ↑ применяем b
```

Фиксировать часть аргументов, получить новую функцию.

```ts
// каррирование
const multiply = (a: number) => (b: number) => a * b;
const double = multiply(2);    // зафиксировали a=2
const triple = multiply(3);    // зафиксировали a=3

double(5);  // 10
triple(5);  // 15

// на практике чаще через замыкание
function createLogger(prefix: string) {
    return (msg: string) => console.log(`[${prefix}] ${msg}`);
}
const dbLog = createLogger('DB');
dbLog('connected'); // [DB] connected
```

## Зачем

Создание специализированных функций из общих. `createLogger('DB')` вместо передачи prefix каждый раз.

## Происхождение

Haskell Curry (1930-е) → JS замыкания.
