---
title: FP Patterns MOC
tags: [moc]
date: 2026-04-12
---

# Паттерны функционального программирования

next: [[interface-vs-discriminated-union]] [[algebraic-data-types]]

---

Паттерны из ФП, которые перетекли в мейнстрим. Отсортированы по практической частоте использования.

## Самые используемые

- [[immutability]] — не мутировать данные, а создавать новые
- [[pure-functions]] — один вход → один выход, без побочных эффектов
- [[higher-order-functions]] — функция принимает или возвращает функцию
- [[option-maybe]] — тип "есть значение или нет" вместо null
- [[result-either]] — тип "успех или ошибка" вместо throw
- [[error-contract-union-vs-throw]] — когда ошибка как значение, когда как канал control flow

## Менее частые, но полезные

- [[function-composition]] — собирать сложное из простых функций
- [[currying]] — фиксировать часть аргументов, получить новую функцию
- [[recursion-vs-loops]] — рекурсия для деревьев и вложенных структур

## Прикладная дисциплина на этих инструментах

- [[type-driven-design]] — Type-Driven Design: программа корректна по построению, не по проверке. Использует ADT, Option/Result, pure/total functions как первичный инструмент дизайна. См. [[type-design-moc]] для всей подборки.

## Что откуда пришло

| Паттерн | Откуда | Когда в мейнстриме |
|---|---|---|
| Иммутабельность | Лямбда-исчисление (1930-е) | React/Redux (2015+) |
| Чистые функции | Haskell (1990) | Повсюду |
| Функции высшего порядка | Lisp (1958) | JS `map/filter` (ES5, 2009) |
| Option/Maybe | ML (1973) | Rust, Swift, Kotlin (2010-е) |
| Result/Either | ML (1973) | Rust (2015), Go `(val, err)` |
| Pattern matching + ADT | ML (1973) | Rust, TS, Kotlin (2010-е) |
| Композиция | Математика | Unix pipes, lodash/fp |
| Каррирование | Haskell Curry (1930-е) | JS замыкания |
