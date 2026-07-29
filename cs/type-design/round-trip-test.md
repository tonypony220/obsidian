---
title: Round-Trip Test
tags: [concept, pattern, testing, property-test]
date: 2026-06-11
---

# Round-Trip Test

moc: [[type-design-moc]]
back: [[parse-dont-validate]]
next:
- [[pin-by-construction]]
- [[zod]]
- [[exhaustiveness-check]]
- [[algebraic-data-types]]
- [[make-illegal-states-unrepresentable]]

---

```
       encode             decode
   x ─────────► bytes ─────────► x'
                                  │
                                  ▼
                          проверка: x == x'

   если ≠ → потеря/искажение по пути
   если = → encode и decode взаимно обратны
```

**TL;DR:** Runtime-пин двух независимых деклараций одной формы (canonical `type` ↔ wire-schema). Прогоняем значение через `encode → decode`, ожидаем эквивалент; расхождение = drift. Нужен только когда одна декларация невозможна (`z.infer` / Rust `derive` не подходят). С `satisfies Record<AllKinds, T>` расширяется до compile-time coverage по дискриминатору.

## Базовая идея

Round-trip — общий property-тест для **биекций**: пары функций, где `decode(encode(x)) == x`. Универсальный приём, не специфичный для контрактов:

| Кодек | Round-trip |
|---|---|
| JSON | `JSON.parse(JSON.stringify(x))` ≟ `x` |
| gzip | `decompress(compress(bytes))` ≟ `bytes` |
| шифрование | `decrypt(encrypt(msg, k), k)` ≟ `msg` |
| base64 | `b64Decode(b64Encode(buf))` ≟ `buf` |
| ISO 8601 | `new Date(d.toISOString())` ≟ `d` |

Везде свойство: цепочка преобразований **не теряет информацию**.

## Применение к контрактам

HTTP-граница — это пара encode/decode:

```
клиент                                          сервер
──────                                          ──────
T (typed object)                                T' (typed object)
  │                                                  ▲
  │ JSON.stringify                                   │ Schema.parse
  ▼                                                  │
"{...}"  ──── HTTP ────►  "{...}"  ───parse JSON───►
(encode)                  (wire)                  (decode)
```

Контракт «работает» ↔ `T = T'`. Round-trip-тест это и проверяет:

```ts
const original: OrderContext = { kind: "offer", offerId: "abc" }
const decoded = Schema.parse(JSON.parse(JSON.stringify(original)))
expect(decoded).toEqual(original)
```

## Что именно ломается

TypeScript не связывает `type T = ...` и `const S = z.object(...)` — это две **независимые** декларации. Компилятор молчит, даже если они разъехались. Round-trip ловит расхождение в CI.

### 1. Поле забыто в схеме → тихо теряется на границе

```ts
type OrderContext = { kind: "offer"; offerId: string }
const Schema = z.object({ kind: z.literal("offer") })   // offerId забыли

const x: OrderContext = { kind: "offer", offerId: "abc" }
const y = Schema.parse(JSON.parse(JSON.stringify(x)))
// y = { kind: "offer" }   ← offerId вырезан Zod'ом (strip по умолчанию)
expect(y).toEqual(x)   // FAIL: missing offerId
```

Реальный класс бага: клиент шлёт `{kind, offerId}`, сервер парсит и получает `{kind}`, `offerId` — `undefined` в handler'е. Падение или тихое неверное поведение.

### 2. Тип разъехался (number ↔ string)

```ts
type OrderContext = { price: number }
const Schema = z.object({ price: z.string() })   // кто-то поменял на string

Schema.parse(JSON.parse(JSON.stringify({ price: 100 })))
// ZodError: Expected string, received number
```

Реальный класс бага: PR «amounts should be strings for precision» обновил схему, canonical `type` остался. Каждый запрос падает на границе.

### 3. Забыли ветвь union'а

```ts
type OrderContext =
  | { kind: "offer" }
  | { kind: "listing" }
  | { kind: "auction" }   // добавили позже

const Schema = z.discriminatedUnion("kind", [
  z.object({ kind: z.literal("offer") }),
  z.object({ kind: z.literal("listing") }),
  // auction забыт
])
```

Простой round-trip поймает `auction` только если тест на него написан. Автоматизацию даёт `satisfies Record<AllKinds, T>` в таблице cases — расширение union без строчки фейлит **компилом**, не рантаймом (см. ниже).

## Coverage по дискриминатору

Round-trip одного значения проверяет **этот вариант**, не весь [[algebraic-data-types|union]]. Для union'а нужен round-trip каждого варианта:

```ts
const cases = {
  listing: { kind: "listing" },
  offer:   { kind: "offer", offerId: "abc" },
  auction: { kind: "auction" },
}

it.each(Object.entries(cases))("round-trip: %s", (_, original) => {
  const decoded = Schema.parse(JSON.parse(JSON.stringify(original)))
  expect(decoded).toEqual(original)
})
```

По одному канону на kind. Схема не знает один из вариантов — соответствующий тест краснеет с точным указанием какой.

## Self-enforcement через `satisfies Record<AllKinds, T>`

Проблема: таблица `cases` пишется руками. Расширили union — забыли добавить строчку — тест не покрывает новый вариант, но остаётся зелёным.

Решение — заставить компилятор требовать полноту таблицы:

```ts
const cases = {
  listing: { kind: "listing" },
  offer:   { kind: "offer", offerId: "abc" },
  auction: { kind: "auction" },
} satisfies Record<OrderContext["kind"], OrderContext>
```

Разбор:
- `OrderContext["kind"]` → union literal `"listing" | "offer" | "auction"`
- `Record<K, V>` → объект, **все** ключи из `K` обязательны
- `satisfies X` → правое выражение обязано удовлетворять типу `X`

Завтра добавили `"preorder"` в `OrderContext`:

```
OrderContext["kind"]  →  + "preorder"
Record<..., T>        →  требует ключ preorder
cases                 →  ключа нет
satisfies             →  ❌ TS error в файле теста
```

Файл теста не компилится → невозможно расширить union и не дополнить тест. Аналог [[exhaustiveness-check|exhaustiveness через `never`]], но для тестов вместо `switch`. Compile-time coverage поверх runtime expectations.

## Что round-trip-тест НЕ покрывает

| Класс бага | Ловит? |
|---|---|
| Схема не знает kind | ✅ |
| Схема искажает значение | ✅ |
| Автор расширил union, забыл схему | ✅ (через `satisfies`) |
| Прод-код вообще не вызывает схему | ❌ |
| Прод-builder шлёт неправильный payload | ❌ |
| Кнопка не дёргает builder | ❌ |
| Drift между deployed server и installed client | ❌ |

Round-trip — это **тест левого края** (схема ↔ canonical type) в изоляции. Цепочку «реальная кнопка → реальный builder → реальная схема → реальный route» проверяет walking skeleton — другой class теста.

## Round-trip как runtime-пин

Round-trip — форма [[pin-by-construction|пина]], только не через тип, а через тест. Тот же принцип «зафиксировать A к B», просто на другом слое enforcement.

| Уровень пина | Ловится | Когда доступен |
|---|---|---|
| type-level (by construction) | компилятор, 0 сек | одна декларация возможна (`z.infer`, Rust `derive`) |
| **runtime (round-trip)** | **CI тест, минуты** | **две декларации, слить нельзя** |
| process (review checklist) | человек, дни | нет ни того, ни другого |

Каждая ступень вниз: ×N дороже поймать, ×N чаще пропущено. Round-trip — компромисс когда свернуть в один тип не получается, но оставлять на review нельзя.

## Когда round-trip не нужен: одна декларация

Идеальный кейс — источник правды один, вторая форма выведена автоматически. Тогда drift'а между декларациями не существует, round-trip избыточен для этого класса бага.

**TS через `z.infer`:**

```ts
const Schema = z.object({ kind: z.literal("offer"), offerId: z.string() })
type OrderContext = z.infer<typeof Schema>   // ← type выведен из schema
```

Одна декларация → drift невозможен by construction. Это уже [[pin-by-construction]], round-trip не нужен.

**Rust через serde derive:**

```rust
#[derive(Serialize, Deserialize, PartialEq)]
struct OrderContext {
    kind: String,
    offer_id: String,
}
```

Одна struct, оба `impl` генерятся макросом. Round-trip тест в Rust всё ещё пишут, но для **других** классов бага:
- кастомные `impl Serialize/Deserialize` без derive → снова две декларации
- schema evolution (v1 wire ↔ v2 canonical)
- cross-format (JSON ↔ protobuf через ту же struct)
- property tests через `proptest`/`quickcheck` над сгенерированными входами

**Стек ↔ нужен ли round-trip для drift:**

| Стек | Деклараций формы | Round-trip для drift? |
|---|---|---|
| TS + `z.infer<typeof S>` | 1 | ❌ невозможен |
| TS + ручной `type` + отдельная `z.object` | 2 | ✅ обязателен |
| Rust `#[derive(Serialize, Deserialize)]` | 1 | ❌ невозможен |
| Rust с ручными `impl` | 2 | ✅ обязателен |
| Protobuf-generated (.proto → сгенерированный код) | 1 | ❌ невозможен |

Т.е. round-trip — **не** особенность TS, а особенность любого стека с раздельными декларациями формы. TS + Zod так делают часто, Rust + derive почти никогда.

## Связь с [[parse-dont-validate]]

Round-trip — property-проверка корректности `parse`-функции на границе:

```
parse(encode(x)) ≟ x   для каждого x ∈ canonical type
```

Если `parse` — это PDV-функция, round-trip пинит: **все** валидные значения canonical типа должны успешно пересечь границу. Без потерь, без отказов.

## Не только в TS / контрактах

| Домен | Round-trip как property |
|---|---|
| Сериализаторы (serde, protobuf, JSON) | `T → bytes → T'` ≟ `T` |
| Парсеры компиляторов | `AST → format(AST) → parse → AST'` ≟ `AST` (pretty-printer property) |
| Криптография | `decrypt(encrypt(m, k), k)` ≟ `m` |
| Базы данных | `read(write(row))` ≟ `row` (схема не теряет поля) |
| Миграции | `down(up(state))` ≟ `state` (reversible migration) |
| Property-based testing (QuickCheck, fast-check) | round-trip — каноничный property шаблон |

Общее везде: **encode и decode — пара взаимно обратных функций**, и тест на это свойство универсален. Любая пара «преобразование туда / преобразование обратно» — кандидат на round-trip.
