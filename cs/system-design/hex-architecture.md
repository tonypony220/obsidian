---
title: Hexagonal Architecture (Ports & Adapters)
tags: [architecture, pattern, hex, di]
date: 2026-06-07
---

# Hexagonal Architecture

back: [[architecture-stack]]
next:
- [[driving-vs-driven-ports]]
- [[seam]]
- [[clean-architecture]]
- [[functional-core-imperative-shell]]
- [[module-organization-patterns]]
- [[middleware-patterns]]

---

```
              ┌──────────────┐
              │   HTTP API   │  ← adapter
              └──────┬───────┘
                     │ port (interface)
   ┌─────────┐       ▼        ┌──────────┐
   │  Tests  │──▶ ╔══════╗ ◀──│   CLI    │
   │ adapter │    ║      ║    │ adapter  │
   └─────────┘    ║DOMAIN║    └──────────┘
                  ║      ║
   ┌─────────┐    ╚══════╝    ┌──────────┐
   │Postgres │──▶          ◀──│  Stripe  │
   │ adapter │       ▲        │ adapter  │
   └─────────┘       │        └──────────┘
                     │ port
              ┌──────┴───────┐
              │ Email/Queue  │ ← adapter
              └──────────────┘
```

**TL;DR:** домен в центре объявляет порты (интерфейсы) в своих терминах; адаптеры снаружи реализуют их под конкретные технологии. Стрелки импортов идут только снаружи внутрь — домен ничего не знает о HTTP, БД, SDK.

## Проблема, которую решает

Без hex бизнес-логика прибита к технологиям:

```typescript
// плохо — логика заказа намертво связана со Stripe и Postgres
async function placeOrder(req, res) {
  const stripe = new Stripe(process.env.STRIPE_KEY)
  const db = new Pool(...)
  const charge = await stripe.charges.create({ amount: req.body.total, ... })
  await db.query('INSERT INTO orders ...', [...])
  res.json({ ok: true })
}
```

Следствия:
- Нельзя протестировать без живого Stripe и Postgres
- Замена Stripe → PayPal требует переписать бизнес-логику
- Бизнес-правило («сумма = сумма позиций») размазано между HTTP-роутом и БД
- Логика существует только внутри Express — не запустить из CLI/cron/очереди

## Решение: порты и адаптеры

**Порт** — интерфейс, который домен объявляет в своих терминах (`PaymentGateway.charge(money)`, не `stripe.charges.create`).

**Адаптер** — реализация порта под конкретную технологию. Переводчик между языком домена и языком SDK.

```typescript
// domain/ports.ts — порт (домен говорит, что ему нужно)
export interface PaymentGateway {
  charge(amount: Money, customer: CustomerId): Promise<ChargeResult>
}

// domain/order-service.ts — домен (знает только порт)
export class OrderService {
  constructor(private payments: PaymentGateway) {}
  async place(order: Order) {
    await this.payments.charge(order.total, order.customerId)
  }
}

// adapters/stripe-gateway.ts — адаптер (знает обе стороны)
import Stripe from 'stripe'
import type { PaymentGateway } from '../domain/ports'

export class StripeGateway implements PaymentGateway {
  constructor(private stripe: Stripe) {}
  async charge(amount: Money, customer: CustomerId) {
    const pi = await this.stripe.paymentIntents.create({
      amount: amount.cents,
      currency: amount.currency.toLowerCase(),
      customer: customer.value,
    })
    return { id: pi.id, status: pi.status === 'succeeded' ? 'ok' : 'pending' }
  }
}
```

## Направление зависимости

Проверка hex через grep:

```
grep -r "from 'stripe'" domain/    → пусто
grep -r "from 'pg'"     domain/    → пусто
grep -r "from 'next'"   domain/    → пусто
```

Если что-то находится — утечка, домен прибит к технологии. Стрелка импорта `StripeGateway → PaymentGateway` идёт снаружи внутрь — это и есть **Dependency Inversion**.

Тонкость: инверсия (и interface) нужна только на driven-стороне — где домен зовёт внешний мир. На driving-стороне (HTTP → домен) портом служит сама сигнатура функции — [[driving-vs-driven-ports]].

## DI как механизм

Домен объявил порт, но конкретный адаптер ему надо как-то получить — не через `new StripeGateway()` (тогда импорт SDK утечёт в домен). Передаём снаружи:

```typescript
// composition root — единственное место, знающее про всех
const stripe = new Stripe(process.env.STRIPE_KEY!)
const payments = new StripeGateway(stripe)
const orders = new OrderService(payments)   // ← DI

// в тесте — тот же домен, другой адаптер
const fakePayments: PaymentGateway = {
  charge: async () => ({ id: 'fake', status: 'ok' }),
}
const orders = new OrderService(fakePayments)
```

DI = техника, которая позволяет соблюдать правило «зависимости только внутрь». Без DI пришлось бы импортировать конкретику в домене.

## Почему «hexagonal»

Алистер Кокбёрн (2005) специально выбрал шестиугольник вместо привычного «слоёного пирога» (UI сверху, БД снизу). Стороны равнозначны — нет верха/низа, любой адаптер (UI, тест, CLI, очередь) подключается одинаково. Число 6 произвольное; сам Кокбёрн потом жалел, что не назвал паттерн просто **Ports & Adapters**.

## Когда применять

| Сложность | Hex |
|---|---|
| CRUD-скрипт, одна таблица | оверкилл |
| Логика > чем «прочитать и записать» | да |
| Меняются провайдеры (платёжки, email, БД) | да |
| Нужны интеграционные тесты без живых сервисов | да |
| Один и тот же домен дёргается из HTTP + cron + CLI | да |

## Не только в коде

Тот же паттерн встречается везде, где есть «ядро + сменные интерфейсы»:

- **Электрическая розетка** — стандартный порт (220V/50Hz), любой прибор-«адаптер» подключается
- **USB** — хост объявил протокол, устройства реализуют
- **Драйверы ОС** — ядро объявило API файловой системы, конкретные ФС (ext4, NTFS, APFS) реализуют
- **JDBC/ODBC** — приложение зовёт стандартный интерфейс, драйвер переводит в диалект конкретной СУБД
- **GraphQL resolvers** — схема = порт, resolver = адаптер к источнику данных

**Общее везде:** стабильный контракт в центре, сменные реализации по периметру, направление зависимости — от реализаций к контракту, не наоборот.
