---
title: Zod
tags: [tool, pattern, type-system, runtime-validation]
date: 2026-06-07
---

# Zod

moc: [[type-design-moc]]
back: [[parse-dont-validate]]
next:
- [[branded-types]]
- [[algebraic-data-types]]
- [[make-illegal-states-unrepresentable]]
- [[exhaustiveness-check]]
- [[round-trip-test]]

---

```
один файл схемы
       │
       ▼
┌───────────────────────────┐
│  const S = z.object({…})  │
└─────────────┬─────────────┘
              │
   ┌──────────┴──────────┐
   ▼                     ▼
compile-time           runtime
type T = z.infer<S>    S.parse(unknown) → T | throw ZodError
   │                     │
доступен в IDE/tsc     работает на границе (route, fetch, webhook)
```

**TL;DR:** Zod — TS-библиотека, где **одна декларация схемы** даёт и статический тип (`z.infer`), и runtime-парсер (`.parse`). Конкретная реализация [[parse-dont-validate]] для TypeScript: schema = single source of truth, дальше типы гарантируют без повторной валидации.

## Что это

```ts
const CheckoutIntent = z.object({
  cartId: z.string().uuid(),
  currency: z.enum(['USD', 'EUR', 'GBP']),
  amount: z.number().int().positive(),
})
type CheckoutIntent = z.infer<typeof CheckoutIntent>  // ← тип бесплатно

const parsed = CheckoutIntent.parse(await req.json())  // throws на mismatch
//    ^ типизирован как CheckoutIntent
```

Два режима, одна декларация:
- **compile-time** — `z.infer<typeof S>` выдаёт TS-тип
- **runtime** — `.parse(unknown)` валидирует и сужает тип; `.safeParse()` возвращает `Result`-shape ([[result-either]])

## Связь с Parse Don't Validate

`.parse()` — буквальная реализация PDV: `unknown → Typed | throw`. Граница: route handler, fetch response, webhook, queue consumer. Дальше по коду — только типизированный `T`, повторно не проверяем.

| | Голые TS-типы | Zod-схема |
|---|---|---|
| compile-time check | ✅ | ✅ (через `z.infer`) |
| runtime check на границе | ❌ (`as T` — ложь) | ✅ (`.parse()` падает) |
| single source of truth | тип отдельно, валидация отдельно | одна декларация |
| ловит unexpected payload | ❌ | ✅ с указанием поля |

## ADT через z.discriminatedUnion

Zod-схема для [[algebraic-data-types]] — sum type через дискриминатор:

```ts
const Order = z.discriminatedUnion('status', [
  z.object({ status: z.literal('pending') }),
  z.object({ status: z.literal('paid'),      capturedAt: z.string() }),
  z.object({ status: z.literal('cancelled'), reason:     z.string() }),
])
type Order = z.infer<typeof Order>

// в коде — exhaustiveness через never ([[exhaustiveness-check]])
function render(o: Order) {
  switch (o.status) {
    case 'pending':   return …
    case 'paid':      return …
    case 'cancelled': return …
    default: { const _: never = o; throw new Error('unreachable') }
  }
}
```

Тогда `z.discriminatedUnion` парсит payload и гарантирует, что `status` — одно из трёх известных. Расширил schema — компилятор требует все `switch` обновить. См. [[make-illegal-states-unrepresentable]].

## Где ставить .parse()

| Место | Парсить? |
|---|---|
| HTTP route handler — входящий request | ✅ `Request.parse()` |
| HTTP route handler — исходящий response | ✅ опц., ловит баги бэка |
| fetch wrapper — пришедший response | ✅ `Response.parse()` |
| webhook consumer | ✅ |
| queue / SQS / Pub/Sub consumer | ✅ |
| внутренний reducer / selector | ❌ — уже типизировано |
| server action → DB | ❌ — internal code |

Принцип PDV: **один раз на границе** во well-typed форму, дальше типы.

## Shared schema между клиентом и сервером

Типовой кейс: один пакет (`packages/contract`) экспортирует schema, web и mobile импортируют её же.

```
packages/contract/src/checkout.ts   ← ОДИН SoT
   │
   ├── apps/web/api/checkout/route.ts     → S.parse(req.body)
   └── apps/mobile/lib/api/checkout.ts    → S.parse(await res.json())
```

Что это даёт:
- **Compile-time:** правка schema → `tsc` падает у всех импортёров одним проходом. Mobile не может быть «со старой схемой» в исходниках.
- **Runtime:** установленный у юзера mobile — это уже скомпилен и крутится. Если задеплоенный web ушёл вперёд, `.parse()` на границе ловит mismatch → 400 ZodError с `path` и `message` вместо silent `undefined` глубоко в коде.

## Что Zod НЕ решает

- **Drift между deployed server и installed client** — нерешаемо без force-upgrade gate. Zod **детектирует** и **канализирует** drift (400 + явный лог), но не предотвращает.
- **Бизнес-инварианты** между полями — Zod проверяет shape; «amount должен соответствовать сумме line items» — это уже domain check после парсинга.
- **Изменение типа без переименования** — если schema требовала `number`, бэк прислал `string`, `.parse()` падает. Но если оба `string` с разным семантическим смыслом — schema не различит. Лечится [[branded-types]].

## Не только в TS

Та же идея «schema = SoT для типа + runtime-парсера» в других экосистемах:

- **Python** — `pydantic`, `attrs` + `cattrs`: декларация класса → автоматический валидатор JSON → typed object
- **OCaml / ReScript** — `ppx_deriving` / `decco`: derive json-кодеков из объявления типа
- **Rust** — `serde` + `#[derive(Deserialize)]`: типы annotated, парсер генерится компилятором
- **Go** — `encoding/json` + struct tags; слабее, т.к. нет ADT — illegal states остаются представимыми
- **Protobuf / gRPC** — `.proto` файл = schema, codegen → типы + парсер на N языков; cross-language вариант той же дисциплины
- **JSON Schema / Ajv** — schema-first, language-agnostic; типы выводятся отдельно (json-schema-to-typescript)
- **GraphQL** — SDL schema → codegen типов + рантайм-валидация на резолверах
- **TS-альтернативы** — `io-ts` (functional, Either), `valibot` (tree-shakeable), `arktype` (TS-syntax-как-schema), `effect/schema` (под Effect runtime)

Общее везде: **граница системы парсит unknown во well-typed форму через одну декларацию**, которая одновременно — спецификация и исполняемый валидатор. Это устраняет drift между «как мы думаем что устроены данные» (тип) и «как мы их реально проверяем» (валидатор).
