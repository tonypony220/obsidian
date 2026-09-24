---
title: Type-Driven Design
tags: [concept, type-system, discipline]
date: 2026-06-07
---

# Type-Driven Design (TyDD)

moc: [[type-design-moc]]
next:
- [[make-illegal-states-unrepresentable]]
- [[total-vs-partial-functions]]
- [[parse-dont-validate]]
- [[branded-types]]
- [[type-design-without-adt]]
- [[type-driven-vs-ddd-tdd-spec-api]]

---

```
        корректность по проверке              корректность по построению
        ──────────────────────                ─────────────────────────
        types: позиции байт                   types: модель домена
        validation: на каждом стыке           parse: один раз на границе
        invariants: в head + tests            invariants: в типах
        новое состояние: grep + надежда       новое состояние: компилятор красный везде
```

**TL;DR:** дисциплина проектирования, в которой типы — первичный инструмент дизайна, а не документация. Три столпа: ADT для моделирования вариантов, illegal states unrepresentable для инвариантов, total functions + Parse Don't Validate для границ. Цель — компилятор отвергает то, что бизнес-логически невозможно.

## Три столпа

1. **Моделирование вариантов через [[algebraic-data-types]]** — sum types для «одно из», product для «всё вместе». Один канонический union на домен, импортируется всеми слоями (UI, resolver, persistence).
2. **[[make-illegal-states-unrepresentable]]** — если состояние X бизнес-логически невозможно, оно должно быть **синтаксически непредставимо**. Не «валидируем что не X», а «X нельзя написать».
3. **[[total-vs-partial-functions]] + [[parse-dont-validate]]** — функция определена на всём входе; знание о валидности фиксируется в типе один раз на границе и дальше не теряется.

## Что это меняет на практике

| Задача | Без TyDD | С TyDD |
|--------|----------|--------|
| Добавить вариант в enum | grep по кодовой базе, надежда на тесты | компилятор красный во всех switch'ах ([[exhaustiveness-check]]) |
| Получить «провалидированный email» | bool + дисциплина «не забыть проверить» | тип `Email`, невалидный физически не пролетит ([[branded-types]]) |
| Order в состоянии shipped без paidAt | runtime-инвариант + тесты + код-ревью | сигнатура type не позволяет такое значение собрать |
| Функция возвращает «может быть число» | `number` + знание «иногда NaN» в голове | `Result<number, DivByZero>` или сужение входа на `NonZero` |

## Роль типов

В TyDD типы — **не пост-фактум аннотация на готовый код**, а **первый артефакт дизайна**. Цикл:

1. Сформулировать домен → выписать типы (variants, invariants, boundaries)
2. Попытаться написать функцию → не компилится → расширить/сузить типы
3. Скомпилилось → большая часть «можно ли это представить» уже доказана компилятором

В пределе (Haskell, Idris, Coq) типы могут выражать произвольно сложные утверждения. На практике — TS/Rust/Kotlin уже дают достаточно для большинства бизнес-инвариантов.

## Origin: ML-семейство

Все ключевые приёмы впервые стали стандартной практикой в ML-семействе языков:
- **ADT + pattern matching** — ML (1973)
- **Option/Maybe**, **Result/Either** — ML → Haskell → Rust/Swift
- **«Make illegal states unrepresentable»** — Yaron Minsky, Jane Street, OCaml (2011)
- **«Parse, Don't Validate»** — Alexis King, Haskell-комьюнити (2019)
- **Exhaustiveness в pattern matching** — встроена в ML/Haskell с самого начала

Сейчас TyDD — общая дисциплина, не привязанная к FP-парадигме. Но связь сильная: см. [[fp-patterns-moc]] — иммутабельность, pure functions, sum types и pattern matching — все они инструменты, которыми TyDD удобнее всего пользоваться. Total ⊂ Pure; иммутабельность защищает ADT-инвариант от мутационного протухания.

## Не только в коде

Тот же принцип «корректность по построению» встречается:

- **Парсеры grammar-based** (PEG, ANTLR) — невозможно «случайно вставить» структуру, которой нет в грамматике
- **API-схемы** (OpenAPI, GraphQL, protobuf) — клиент не может собрать запрос несуществующей формы; см. [[type-driven-vs-ddd-tdd-spec-api]]
- **State machine для UI** (XState) — переход, не описанный в FSM, физически не может произойти
- **Linear types / borrow checker** (Rust) — нельзя «случайно» использовать освобождённую память
- **Dependently typed proof assistants** (Coq, Lean) — теорема не может быть «почти доказана»

Общее везде: **структурная невозможность ошибки** заменяет **процедурную проверку** на ошибку.
