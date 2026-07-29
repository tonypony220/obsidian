---
title: Functional Core, Imperative Shell
tags: [architecture, pattern, fp, fc-is, discipline]
date: 2026-06-07
---

# Functional Core, Imperative Shell

back: [[hex-architecture]]
next:
- [[clean-architecture]]
- [[pure-functions]]
- [[immutability]]

---

```
Hex / Clean (структура):           FC/IS (дисциплина внутри):

  ┌── adapters ──┐                  shell.load()         ← IO до
  │              │                       ↓
  │  ┌─ domain ┐ │                  core(data)           ← чистая функция
  │  │ зовёт   │ │                       ↓
  │  │ порты   │ │                  shell.save(result)   ← IO после
  │  └─────────┘ │                       ↓
  └──────────────┘                  shell.notify(...)    ← IO после

  где живут файлы                  что разрешено домену делать
  (структурное правило)            (поведенческое правило)
```

**TL;DR:** FC/IS — это **дисциплина**, а не альтернативная архитектура: домен = только чистые функции `данные → данные`, никакого IO внутри. Ортогонально [[hex-architecture]] и [[clean-architecture]] — те задают **структуру** (где файлы, направление импортов), FC/IS задаёт **поведение** (что домен может делать). Профит виден только когда ядро тяжёлое, IO — лёгкая обвязка.

## Идея

В hex домен общается с внешним миром через порты — то есть всё-таки общается, через `await this.repo.save()`. FC/IS идёт дальше: **домен не зовёт ничего**. Совсем. Никаких портов, никаких `async`. Всю грязь (IO, время, случайность, БД, сеть) делает **оболочка** — снаружи, **до** и **после** вызова ядра.

Сформулированная Gary Bernhardt (Boundaries talk, 2012). Иногда называется calculation vs action из ФП.

## Минимальный пример где профит виден

Плохой пример (создание заказа — почти вся работа это оркестрация):

```typescript
// core
export function createOrder(input, now): Order { /* 5 строк */ }
// shell
async function place(input) {
  const order = createOrder(input, new Date())
  await db.save(order)
  await email.send(...)
}
```

FC/IS тут не нужен — Clean покрыл бы то же самое. Ядро тривиально, профит ноль.

Хороший пример (pricing — реальные правила):

```typescript
// core/pricing.ts — сотни строк правил, чистая функция
export function priceCart(
  cart: Cart, rules: PricingRules, customer: Customer, region: Region,
): PriceBreakdown {
  const subtotal      = sumItems(cart.items)
  const tierDiscount  = applyTierDiscount(subtotal, customer.tier, rules)
  const promoDiscount = cart.promoCode ? applyPromo(subtotal, cart.promoCode, rules, customer) : null
  const bulkDiscount  = applyBulkRules(cart.items, rules)
  const discounts     = [tierDiscount, promoDiscount, bulkDiscount].filter(Boolean)
  const afterDisc     = subtract(subtotal, sumDiscounts(discounts))
  const tax           = computeTax(afterDisc, region, cart.items)
  const total         = add(afterDisc, tax)
  return { subtotal, discounts, tax, total }
}

// shell — тонкий
async function checkout(cartId, customerId) {
  const [cart, customer, rules] = await Promise.all([
    db.cart.load(cartId), db.customer.load(customerId), db.pricingRules.current(),
  ])
  const breakdown = priceCart(cart, rules, customer, customer.region)   // всё вычисление здесь
  const invoice   = buildInvoice(breakdown, customerId, new Date())     // тоже чистая
  await db.invoices.save(invoice)
  await stripe.charge(invoice.total, customer.stripeId)
  await email.sendReceipt(customer.email, invoice)
  return invoice
}
```

## Профит: тесты без налога моков

```typescript
// FC/IS — чистый ввод-вывод
test('VIP + промо + Германия → 15% скидка + 19% VAT', () => {
  expect(priceCart(vipCart, rules, vipCustomer, 'DE')).toEqual({ ... })
})
```

50 таких тестов покрывают всю таблицу правил, прогоняются за миллисекунды, никаких `beforeEach`, никаких фейк-репозиториев. Hex с портами потребовал бы конструировать фейковые `CartRepo`/`CustomerRepo` для каждого теста — терпимо на 50, ад на 500.

**Это и есть профит**. Не архитектурная красота, а: юнит-тесты ядра не платят налог моков, когда правил много и они комбинируются.

## Ортогональность hex/Clean

FC/IS — не альтернатива hex/Clean, а **дополнительная ось**:

| | Домен зовёт IO (через порты) | Домен чисто функционален |
|---|---|---|
| **Hex/Clean структура** | стандартный hex с `OrderService.place()` | hex с «пустыми» портами — shell зовёт IO, домен только считает |
| **Без формальной структуры** | один файл, всё вперемешку | один файл, но логика собрана в чистые функции |

То есть:
- **Hex** отвечает: «куда положить логику?» → внутрь, не наружу. Структура.
- **Clean** уточняет: «как разделить внутреннее?» → entities vs use cases. Структура.
- **FC/IS** отвечает: «что разрешено логике делать?» → только считать, никаких эффектов. Дисциплина.

Можно сказать: FC/IS — это hex, где порты вырождены до нуля, потому что домен ничего не зовёт.

## Применять локально, не глобально

В реальной кодовой базе обычно сосуществуют оба:

- Большие сценарии — **hex/Clean** с портами (IO переплетена с логикой)
- Отдельные «вычислительные сердца» — **FC/IS внутри**, как pure core, который shell-сценарий зовёт между IO-шагами

То есть **FC/IS — слой дисциплины**, который накладывается поверх любой структуры **точечно**, там где есть тяжёлая логика и тонкая обвязка.

## Где живёт хорошо

- **Reducers** (Redux, `useReducer`): `(state, action) => state` — буквальный FC. Shell = диспатчер/middleware
- **React render**: чистый (`props → UI`), эффекты в `useEffect` — тоже FC/IS
- **Расчёты цен/налогов/скидок**: много правил, мало IO
- **State machines** для заказа/подписки: `(state, event) → newState | error`
- **Парсеры/компиляторы**: текст → AST → байткод
- **ML-пайплайны**: загрузил датасет → серия чистых трансформов → сохранил
- **Валидаторы**: данные → `Result<Valid, Errors>` ([[result-either]])

## Где плохо

- **Workflows с условным IO**: «загрузить X, если стоимость > Y». FC/IS вынуждает либо грузить заранее (расточительно), либо дробить сценарий — это уже Clean с use cases
- **Длинные транзакции с compensating actions**: shell перестаёт быть тонким
- **Стриминг**: core оперирует данными целиком; если их терабайт — не получится

## Эвристика выбора

| Соотношение в сценарии | Что брать |
|---|---|
| Много IO, мало вычислений | [[clean-architecture]] / [[hex-architecture]] |
| Мало IO, много вычислений | FC/IS (поверх hex или просто) |
| Поровну | Clean; FC/IS внутри отдельных тяжёлых функций |

## Не только в коде

Тот же паттерн — отделение **вычисления** от **действия** — встречается везде:

- **Haskell**: монада `IO a` отделяет эффекты от чистых вычислений типом
- **Elm**: `update : Msg -> Model -> (Model, Cmd Msg)` — `update` чистый, `Cmd` это «попросить runtime сделать IO»
- **Redux**: reducers чистые, side-effects в middleware (thunks, sagas)
- **React**: render — чистый, эффекты в `useEffect`
- **Database engines**: query planner — чистая функция (`SQL → plan`), executor — грязный
- **Git**: object model (commits/trees/blobs) — immutable данные, working tree — грязный shell

**Общее везде:** ядро превращает входные данные в выходные **детерминированно**, оболочка отвечает за «откуда данные пришли» и «куда поедут». Граница чёткая — ядро тестируется без оболочки.
