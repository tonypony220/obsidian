---
title: Architecture stack — слои паттернов по cost-of-change
tags: [architecture, concept]
date: 2026-06-11
---

# Architecture stack

next:
- [[type-design-moc]]
- [[seam]]
- [[hex-architecture]]
- [[test-strategy-moc]]
- [[enforcement-moc]]
- [[engineering-process-moc]]

---

```
дёшево ◄──────── стоимость изменения / обнаружения ошибки ────────► дорого

│ контракт    │ домен      │ верификация    │ e2e + obs    │ процесс    │
│ schema, ADT │ hex, lib/  │ skeleton, unit │ logs, alerts │ gates, PM  │
├─────────────┼────────────┼────────────────┼──────────────┼────────────┤
│ компилятор  │ unit-тест  │ integration/CI │ prod-сигнал  │ люди       │ ← кто ловит
├─────────────┼────────────┼────────────────┼──────────────┼────────────┤
│ пины, brand │ fence      │ fitness gate   │ alert        │ merge gate │ ← замок слоя
└─────────────┴────────────┴────────────────┴──────────────┴────────────┘
                  под всеми слоями: 4 силы (см. ниже)
```

**TL;DR:** архитектурная дисциплина = удержание стоимости изменения ~константной при росте системы. Паттерны — не каталог: каждый живёт на слое стека, выводится из 4 сил и работает только в связке. Внедрять и чинить — слева направо.

## Ось: cost of change

- Слои упорядочены по стоимости обнаружения ошибки: компилятор → unit → integration → e2e → prod → инцидент. Каждый шаг вправо ≈ ×10 (Boehm 1981; сегодня «shift-left»)
- Главное правило: инвариант выталкивается в самый левый слой, который способен его удержать
- Чинить тоже слева направо: правое не чинит левое

## Слои

| Слой | Что живёт | Заметки |
|---|---|---|
| Контракт | canonical schema (SSOT), discriminated unions + never-probe, boundary parse (consumer) + producer-пины, brand gate | [[type-design-moc]], [[parse-dont-validate]], [[exhaustiveness-check]], [[branded-types]], [[round-trip-test]], [[pin-by-construction]] |
| Домен | hex: чистые решения в `lib/`, canonical builders (extend-don't-fork), тонкие routes | [[hex-architecture]], [[functional-core-imperative-shell]], [[clean-architecture]] |
| Верификация | walking skeleton, baseline-positive на реальном builder'е, boundary-reject, fitness functions; правее — минимальный e2e, structured logs, alerts | [[test-strategy-moc]] |
| Процесс | branch/merge gate, postmortem → class sweep, audit by endpoint, spec-first | [[engineering-process-moc]] |

- **Enforcement — не слой, а колонка** ([[enforcement-moc]]): у каждого слоя свой замок (контракт — пины/brand gate; домен — eslint fence + allowlist/ratchet; верификация — fitness gate в CI; процесс — merge gate, [[hard-gate-silent-nack]] в CLAUDE.md)
- Проектный инстанс: `ARCHITECTURE_STACK.md` (kraahl) — эти же слои как L0–L6: L0 = принципы/силы, L4 frontend wiring = проекция домен+верификация+fence на фронтовую поверхность; плюс progress-таблица по areas и recipe внедрения

## 4 силы (зачем каждый паттерн существует)

| Сила | Суть | Канон |
|---|---|---|
| SSOT | факт объявлен ровно один раз; дубль = гарантированный drift | DRY (Pragmatic Programmer, 1999) |
| Left shift | ловить максимально слева; ×10 за шаг вправо | Boehm 1981, shift-left testing |
| Enforcement | правило без механизма = пожелание | SE at Google: not enforced = not real |
| Эволюция | внедрять инкрементально, мир не останавливается | strangler (Fowler), ratchet, fix-on-touch |

## Связки: паттерн без пары — полумера

- Ядро + замок: builder (SSOT) без brand gate / fence — пожелание; замок без ядра — friction без ценности → **ядро раньше замка**
- Union + never-probe: без probe новый kind молча проскальзывает мимо switch'ей
- Boundary parse + producer-пины: проверенная одна сторона провода не гарантирует другую
- Fence + allowlist + ratchet + fix-on-touch: иначе fence на legacy-репо не включить
- Skeleton + пины: skeleton проверяет «наш код до провода», пины — «провод соответствует контракту»

## Порядок внедрения

contract → hex → builders → пины → тесты → fence/brand/ratchet. Следствие сил: правые опираются на левые (skeleton'у нужен hex; fence'у нужен canonical путь, который он охраняет). Сколько слоёв оплачивать — [[calibration]]: не слой и не паттерн, а регулятор глубины стека; критерий per layer = cost-of-being-wrong × вероятность × 1/reversibility, не размер проекта.
