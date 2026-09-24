---
title: Clean Architecture
tags: [architecture, pattern, clean, di]
date: 2026-06-07
---

# Clean Architecture

back: [[hex-architecture]]
next:
- [[use-cases-as-named-operations]]
- [[middleware-patterns]]

---

```
        ┌─────────────────────────────────────┐
        │  Frameworks & Drivers               │  ← Next.js, Express, CLI, cron
        │  ┌───────────────────────────────┐  │
        │  │  Interface Adapters           │  │  ← Postgres repo, Stripe gateway
        │  │  ┌─────────────────────────┐  │  │
        │  │  │  Use Cases              │  │  │  ← PlaceOrder, CancelOrder
        │  │  │  ┌───────────────────┐  │  │  │
        │  │  │  │  Entities         │  │  │  │  ← Order, Money, бизнес-правила
        │  │  │  └───────────────────┘  │  │  │
        │  │  └─────────────────────────┘  │  │
        │  └───────────────────────────────┘  │
        └─────────────────────────────────────┘
                     зависимости →
                     только внутрь
```

**TL;DR:** Clean = [[hex-architecture]] + явный слой Use Cases между entities и адаптерами. Hex кладёт всё «внутреннее» в один мешок; Clean делит на «правила домена» (entities) и «сценарии приложения» (use cases).

## Проблема, которую решает hex

В hex есть «домен» и «адаптеры», но внутри домена не различаются два разных типа кода:

1. **Бизнес-правила**, которые истинны независимо от приложения:
   «Заказ не может быть пустым», «сумма = сумма позиций», «нельзя отменить отгруженный»
2. **Сценарии**, которые специфичны этому приложению:
   «Пользователь создаёт заказ → сохранить в БД → отправить email»

Первое существовало бы, даже если бы сайта не было. Второе — это «как именно мы используем правила в нашем UI».

В небольшом hex-проекте оба живут рядом, и это нормально. Но когда:
- Появляется второй frontend (web + mobile + API для партнёров) — сценарии расходятся, правила — нет
- Растёт количество сценариев — нужно явное место для оркестрации
- Хочется тестировать сценарий целиком («что происходит при создании заказа») отдельно от правил («корректна ли сумма»)

— hex без второго слоя начинает плыть.

## Решение: 4 концентрических круга

Uncle Bob (2012) предложил разнести внутреннее на два кольца. Правило одно — **зависимости только внутрь**.

### Круг 1 — Entities (центр)

Бизнес-правила, истинные вне зависимости от приложения.

```typescript
// entities/order.ts — ничего не импортирует из проекта
export type Money = { cents: number; currency: 'USD' | 'EUR' }
export type Order = {
  id: string
  items: OrderItem[]
  total: Money
  status: 'pending' | 'paid' | 'cancelled'
}

export function createOrder(id: string, items: OrderItem[]): Order {
  if (items.length === 0) throw new Error('Order must have items')
  const total = items.reduce(
    (s, it) => ({ cents: s.cents + it.price.cents * it.qty, currency: it.price.currency }),
    { cents: 0, currency: items[0].price.currency },
  )
  return { id, items, total, status: 'pending' }
}
```

### Круг 2 — Use Cases

Сценарии приложения. Оркеструют entities + дёргают порты для внешнего мира.

```typescript
// use-cases/ports.ts
export interface OrderRepository { save(o: Order): Promise<void> }
export interface EmailSender { sendConfirmation(o: Order, to: string): Promise<void> }

// use-cases/place-order.ts
import { createOrder } from '../entities/order'
import type { OrderRepository, EmailSender } from './ports'

export class PlaceOrder {
  constructor(private orders: OrderRepository, private email: EmailSender) {}

  async execute(input: { id: string; items: OrderItem[]; userEmail: string }) {
    const order = createOrder(input.id, input.items)   // entity
    await this.orders.save(order)                       // через порт
    await this.email.sendConfirmation(order, input.userEmail)
    return order
  }
}
```

### Круг 3 — Interface Adapters

Реализации портов под конкретные технологии. То же, что адаптеры в [[hex-architecture]].

```typescript
// adapters/postgres-order-repo.ts
import type { Pool } from 'pg'
import type { OrderRepository } from '../use-cases/ports'

export class PostgresOrderRepository implements OrderRepository {
  constructor(private db: Pool) {}
  async save(o: Order) {
    await this.db.query('INSERT INTO orders ...', [o.id, o.total.cents, ...])
  }
}
```

### Круг 4 — Frameworks & Drivers (снаружи)

Next.js, Express, CLI, cron — то, что зовёт use cases в ответ на внешнее событие.

```typescript
// app/api/orders/route.ts
import { Pool } from 'pg'
import { PlaceOrder } from '@/use-cases/place-order'
import { PostgresOrderRepository } from '@/adapters/postgres-order-repo'
import { SendGridEmail } from '@/adapters/sendgrid-email'

// composition root
const db = new Pool({ connectionString: process.env.DATABASE_URL })
const placeOrder = new PlaceOrder(
  new PostgresOrderRepository(db),
  new SendGridEmail(),
)

export async function POST(req: Request) {
  const order = await placeOrder.execute(await req.json())
  return Response.json(order)
}
```

## «Снаружи / внутри» в коде

Это не физическое расположение файлов, а **направление импортов**.

| Файл | Что импортирует |
|---|---|
| `entities/order.ts` | ничего из проекта |
| `use-cases/place-order.ts` | entities, ports |
| `adapters/postgres-order-repo.ts` | use-cases (порты), entities + `pg` |
| `app/api/orders/route.ts` | adapters, use-cases + `next` |

Проверка через grep — каждое внутреннее кольцо не должно тянуть внешние технологии:

```
grep -r "from 'pg'"   src/entities/    → пусто
grep -r "from 'pg'"   src/use-cases/   → пусто
grep -r "from 'next'" src/use-cases/   → пусто
```

## Разница с hex одной строкой

- **Hex:** «не клади бизнес-логику в HTTP-роут»
- **Clean:** «не клади бизнес-логику в HTTP-роут, **И** раздели бизнес-правила и сценарии»

Для малого проекта эта вторая граница незаметна — Clean и hex выглядят одинаково. Для большого даёт ось переиспользования: entities можно тащить в другое приложение (web + mobile + bot), use cases обычно нет.

## Когда применять

Триггер — **не количество клиентов**, а расхождение сценариев или дублирование оркестрации. Два фронта с одинаковыми сценариями спокойно сидят на hex.

| Ситуация | Что брать |
|---|---|
| CRUD-скрипт | ни то, ни другое |
| Логика умеренная, оркестрация из одного места | [[hex-architecture]] |
| Один сценарий зовётся из HTTP + cron + очереди + CLI | Clean (use case = единое место оркестрации) |
| Сценарии расходятся при общих правилах (`PlaceOrderByCustomer` vs `ByAdmin` vs `FromPartner`) | Clean |
| Route handler стал «оркестратором на 5+ шагов» | Clean (вынести оркестрацию из доставки) |
| Сложный домен с богатыми правилами | Clean + DDD |

Анти-триггер: «у нас web + mobile с одинаковым флоу» — это hex, не Clean. Use cases один на оба фронта.

## Семейство «изолируй домен»

Clean, hex, Onion (Jeffrey Palermo, 2008), Functional Core / Imperative Shell (Gary Bernhardt) — **один паттерн в разных формулировках**. Все решают: ядро не знает о доставке, зависимости идут внутрь, сменные адаптеры по периметру. Различаются только тем, сколько внутренних колец явно выделяют и какую терминологию используют. Выбирай словарь команды, не саму архитектуру.
