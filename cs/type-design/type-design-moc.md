---
title: Type Design MOC
tags: [moc]
date: 2026-06-07
---

# Type-Driven Design — MOC

back: [[architecture-stack]]
next:
- [[type-driven-design]]
- [[fp-patterns-moc]]

---

Дисциплина проектирования через типы: программа корректна **по построению**, а не **по проверке**. Origin — ML-семейство (OCaml, Haskell, F#), сейчас доступно в любом языке с достаточно выразительной системой типов.

## Зонтик

- [[type-driven-design]] — главная concept-нота, три столпа, роль типов как primary design tool

## Принципы

- [[make-illegal-states-unrepresentable]] — невозможное состояние не должно компилироваться; boolean-flag tangle как главный анти-паттерн
- [[total-vs-partial-functions]] — функция определена на всём своём входе; стратегии сузить вход / расширить выход
- [[parse-dont-validate]] — `parse :: Raw → Typed | Error` вместо `validate :: Raw → bool`; парсить на границе один раз

## Техники

- [[branded-types]] — nominal-типы в structural-системе (`Email`, `OrderId`, `NonZero`)
- [[algebraic-data-types]] — sum + product, базовая алгебра типов
- [[exhaustiveness-check]] — компилятор требует обработать все варианты

## Инструменты

- [[zod]] — TS-библиотека: schema → `z.infer` (тип) + `.parse` (рантайм); конкретная реализация PDV

## Тестирование

- [[round-trip-test]] — property-тест «encode/decode не теряет информацию»; в применении к контракту + `satisfies Record<AllKinds, T>` даёт compile-time покрытие union'а

## Применимость и пересечения

- [[type-design-without-adt]] — TyDD в Go / старой Java / Python: что доступно, что теряется
- [[type-driven-vs-ddd-tdd-spec-api]] — пересечения с DDD, TDD, spec-driven dev, API contract design

## Инструменты из FP, на которых стоит TyDD

- [[option-maybe]] — «есть значение или нет» как ADT
- [[result-either]] — «успех или ошибка» как ADT, инструмент Parse Don't Validate
- [[error-contract-union-vs-throw]] — когда union, когда throw
- [[immutability]] — без неё ADT-инвариант протухает мутацией
- [[pure-functions]] — total ⊂ pure
