---
title: Use Cases как именованные операции (vs if/else, vs полиморфизм)
tags: [architecture, pattern, clean, use-cases]
date: 2026-06-07
---

# Use Cases как именованные операции

back: [[clean-architecture]]
next:
- [[hex-architecture]]
- [[exhaustiveness-check]]

---

```
Anemic / одна функция:              Clean / именованные операции:

  place(input) {                      checkout → PlaceByCustomer    ┐
    if customer ...                   admin    → PlaceByAdmin       ├──▶ createOrder()
    if admin    ...                   webhook  → ImportFromPartner  ┘
    if partner  ...                          ↑
  }                                   выбор на точке входа,
       ↑                              а не в теле функции
  runtime-диспатч
  внутри одного тела
```

**TL;DR:** разные сценарии = разные use cases (отдельные классы со своими DTO), не ветки `if kind===` в одной функции. Выбор сценария уходит **наверх** к точке входа (роут импортирует свой use case). Правила остаются общими в entities, ветвление исчезает, а не маскируется интерфейсом.

## Три ступени эволюции

### 1. Anemic — if/else в роуте

```typescript
export async function POST(req) {
  const body = await req.json()
  if (session.role === 'admin')        { /* ветка админа */ }
  else if (req.headers['x-partner'])   { /* ветка партнёра */ }
  else                                  { /* ветка клиента */ }
}
```

Правила размазаны по веткам, добавление источника = ещё одна ветка, тестировать = поднимать HTTP.

### 2. Hex без use cases — if/else в сервисе

Hex говорит «логика не в роуте» — выносим в `OrderService`. Но ветвление может **переехать**, а не исчезнуть:

```typescript
class OrderService {
  async place(input: PlaceInput) {
    const order = createOrder(input.items, input.customerId)   // правило в entity
    if (input.kind === 'admin')   { /* audit + email */ }
    else if (input.kind === 'partner') { /* идемпотентность + email менеджеру */ }
    else                          { /* email клиенту + analytics */ }
  }
}
```

Лучше anemic (правило в entity, БД через порт), но DTO стал union'ом с опциональными полями («admin требует `reason`, partner требует `externalId`») — типы перестали выражать инварианты.

### 3. Clean — отдельные use cases, выбор на роуте

```typescript
// app/api/checkout/route.ts        ← решает: customer
await placeOrderByCustomer.execute({ customerId, items })

// app/api/admin/orders/route.ts    ← решает: admin
await placeOrderByAdmin.execute({ adminId, customerId, items, reason })

// app/api/partner-webhook/route.ts ← решает: partner
await importOrderFromPartner.execute({ partnerId, externalId, items, contractPrices })
```

Каждый use case — линейная функция, **никакого `if kind===`**. У каждого свой DTO с обязательными полями. Развилка ушла наверх — в роутинг, где она естественна (роутер и так маршрутизирует по URL).

## Это **не** классический полиморфизм

Классический «replace conditional with polymorphism» (Fowler) выглядит так:

```typescript
interface OrderPlacer { execute(input: any): Promise<Order> }
const placer = registry.get(kind)   // runtime-диспатч по типу
await placer.execute(input)
```

В Clean так **не делают**, потому что у use cases **разные DTO**: `PlaceByAdmin` требует `reason`, `ImportFromPartner` — `externalId`. Натянуть на них общий `execute(input: any)` = вернуться к `any` и runtime-проверкам.

Точная формула того, что произошло: **«replace conditional with separate named operations + push the choice up to the caller»**. Ветвление не маскируется интерфейсом — оно исчезает, потому что вызывающий уже знает, какой сценарий ему нужен (это знание выражено URL-ом роута).

## Common shape ≠ common contract

Частая ловушка: три use case выглядят похоже — у всех метод `execute()`, все возвращают `Promise<Order>` — кажется, что просится общий интерфейс. Это **структурное сходство**, не полиморфизм.

Полиморфизм работает на **взаимозаменяемости**: вызывающий код должен мочь подставить любую реализацию, не зная конкретный тип. Проверка:

```typescript
const placer: ??? = pickByKind(kind)
await placer.execute(input)   // ← каким должен быть тип input?
```

Не сходится. `PlaceByCustomer` принимает `{customerId, items}`, `PlaceByAdmin` требует `reason`, `ImportFromPartner` — `externalId`. Если вызывающий **обязан знать** конкретный use case (чтобы собрать правильный DTO), полиморфизма нет — есть три похожих по форме класса.

### Generic-интерфейс ничего не даёт

```typescript
interface UseCase<Input, Output> { execute(input: Input): Promise<Output> }
class PlaceByCustomer implements UseCase<CustomerInput, Order> { ... }
class PlaceByAdmin    implements UseCase<AdminInput, Order>    { ... }
```

- `UseCase<CustomerInput>` ≠ `UseCase<AdminInput>` — в один массив не положишь
- Получатель `UseCase<X>` уже знает `X` → конкретный use case уже выбран
- Контракт «функция от чего-то к чему-то» = пустой, ничего не обещает

Это **тавтология в типах**, не абстракция.

### Аналогия в React

Каждый компонент — это `(props: Props) => ReactNode`. Все структурно одинаковы, но никто не пишет `const c: Component<???> = pickComponent(); c(props)`. Каждый caller знает, что он хочет — `<Header />` или `<Modal />`. Они не полиморфны, они просто функции с общей формой.

### Эвристика «есть ли тут интерфейс»

| Вопрос | Если «да» на оба |
|---|---|
| DTO на входе совпадают (один и тот же тип, не «оба что-то принимают») | |
| Вызывающий может работать, не зная конкретный тип | → полиморфизм, делай порт |

Хотя бы одно «нет» — структурное сходство без полиморфизма, не выдумывай интерфейс. Use cases чаще всего «нет» на оба.

## Когда полиморфизм уместен

Полиморфизм оправдан, когда **контракт реально один**, а реализаций N:

- 8 платёжных провайдеров: `PaymentGateway.charge(money) → result` — у всех идентичный смысл
- 3 email-провайдера: `EmailSender.send(to, subject, body)`
- 5 storage-бэкендов: `BlobStore.put(key, bytes)`

Здесь общий интерфейс честный — это [[hex-architecture]] порты/адаптеры.

## Эвристика

| Сигнал | Что брать |
|---|---|
| Один контракт + N реализаций (Stripe vs PayPal) | **Полиморфизм** (порт + адаптеры) |
| N разных операций со своими DTO и побочками | **Отдельные use cases**, выбор на роуте |
| Один сценарий, разные источники (HTTP+cron+CLI) | **Один use case**, N тонких адаптеров его дёргают |
| Внутри функции `switch(kind)` с разными ветками побочек | Дробить на use cases |

## Связано

- [[clean-architecture]] — где живут use cases в общей картине
- [[exhaustiveness-check]] — если всё-таки делаешь `switch` (например, по статусу одного `Order`), `never`-probe ловит забытые ветки. Use cases — это про то, что часто `switch` вообще не нужен
