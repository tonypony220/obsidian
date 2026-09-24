---
title: Algebraic Data Types (ADT)
tags: [concept, type-system]
date: 2026-04-12
---

# Алгебраические типы данных (ADT)

next:
- [[interface-vs-discriminated-union]]
- [[exhaustiveness-check]]
- [[make-illegal-states-unrepresentable]]
- [[type-driven-design]]

---

```
Sum type (A | B | C)          Product type (A × B × C)
┌─────────────────────┐       ┌─────────────────────┐
│  Shape              │       │  Light               │
│  ├─ Circle(r)       │       │  ├─ on: bool         │
│  ├─ Rect(w, h)      │       │  ├─ color: 'r'|'g'   │
│  └─ Triangle(b, h)  │       │  └─ 2 × 2 = 4 values │
│  2 + 3 + 2 = 7 vals │       └─────────────────────┘
└─────────────────────┘
```

Система построения типов из двух операций — суммы и произведения. Название "алгебраические" — потому что типы складываются и перемножаются как числа.

## Product type (тип-произведение)

"Все поля вместе". Обычный struct/record/object. Количество возможных значений = произведение вариантов каждого поля.

```ts
type Light = { on: boolean; color: 'red' | 'green' }
// on: 2 варианта × color: 2 варианта = 4 комбинации
// { on: true,  color: 'red'   }
// { on: true,  color: 'green' }
// { on: false, color: 'red'   }
// { on: false, color: 'green' }
```

Перемножаем поля — поэтому "произведение".

## Sum type (тип-сумма)

"Одно из". Каждый вариант несёт свои данные. Количество значений = сумма вариантов.

```ts
type Pet =
    | { kind: 'cat'; indoor: boolean }         // 2 комбинации
    | { kind: 'dog'; size: 'S' | 'M' | 'L' }  // 3 комбинации
    | { kind: 'fish' }                          // 1 комбинация
// Всего: 2 + 3 + 1 = 6 возможных значений
```

Складываем варианты — поэтому "сумма".

## Комбинация сумм и произведений

Любая структура данных — комбинация:

```ts
type NavAction =
    | { type: 'SELECT'; profile: Profile }     // произведение (type × profile)
    | { type: 'DESELECT'; profileId: number }  // произведение (type × profileId)
    | { type: 'GO_BACK' }                      // единица
    | { type: 'GO_FORWARD' };                  // единица
// ↑ сумма четырёх произведений
```

## Где поддерживаются

Языки с полноценными ADT: Haskell, OCaml, F#, Rust (`enum`), Swift (`enum` с associated values), Kotlin (`sealed class`), TypeScript (union types).

Языки без sum types эмулируют "одно из" через наследование или ручные теги — но теряют [[exhaustiveness-check]].
