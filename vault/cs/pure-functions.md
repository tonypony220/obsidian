---
title: Pure Functions
tags: [concept, pattern]
date: 2026-04-12
---

# Чистые функции

moc: [[fp-patterns-moc]]
next: [[immutability]] [[higher-order-functions]]

---

```
нечистая:                      чистая:
tax = 0.2 ──┐                  (amount, tax)
             ▼                       │
amount ──► price() ──► ?       ──► price() ──► amount*(1+tax)
           меняется tax?             всегда одинаково
           результат разный          нет внешних зависимостей
```

Один вход → один выход, без побочных эффектов. Один и тот же аргумент всегда даёт один результат.

```ts
// нечистая — зависит от внешнего состояния
let tax = 0.2;
function price(amount: number) {
    return amount * (1 + tax); // результат зависит от tax
}

// чистая — всё через аргументы
function price(amount: number, tax: number) {
    return amount * (1 + tax); // всегда одинаковый результат
}
```

## Где встречается

`Array.map/filter/reduce`, React компоненты (props → JSX), утилитарные функции, трансформации данных.

## Зачем

Легко тестировать (не нужны моки), легко кешировать (`useMemo`), легко рассуждать о коде — смотришь на сигнатуру и знаешь всё.

## Происхождение

Haskell (1990) → повсюду.
