---
title: Driving vs Driven Ports (интерфейс только против стрелки)
tags: [architecture, pattern, hex, di]
date: 2026-08-17
---

# Driving vs Driven Ports

back: [[hex-architecture]]
next:
- [[clean-architecture]]
- [[use-cases-as-named-operations]]
- [[functional-core-imperative-shell]]

---

```
DRIVING-сторона (in)                DRIVEN-сторона (out)
HTTP / UI / CLI / тест              БД / платёжка / почта

route ──import──▶ domain            domain ──call───▶ PayPal (runtime)
route ──call────▶ domain            domain ──import─▶ НИЧЕГО
                                    infra  ──import─▶ domain/ports.ts
стрелки СОВПАДАЮТ                   стрелки ПРОТИВОПОЛОЖНЫ
→ интерфейс не нужен                → интерфейс = инверсия
```

**TL;DR:** интерфейс-порт как отдельная сущность нужен только на driven-стороне, где runtime-вызов направлен наружу против import-стрелки. На driving-стороне обе стрелки смотрят внутрь — портом служит сама сигнатура доменной функции.

## Две стороны гексагона

| | Driving (primary, in) | Driven (secondary, out) |
|---|---|---|
| Кто/кого | внешний мир **зовёт домен** | домен **зовёт внешний мир** |
| Примеры | HTTP-роут, UI-хэндлер, CLI, queue-consumer, тест | БД, платёжный шлюз, email, другой сервис |
| Runtime-стрелка | внутрь (адаптер → домен) | наружу (домен → SDK) |
| Import-стрелка | внутрь (адаптер → домен) | должна быть внутрь ⇒ инверсия |
| Порт = | **сигнатура** use-case функции + её in/out типы | **interface**, объявленный в домене |
| Адаптер = | роут/хэндлер: парсит вход, зовёт домен, шлёт ответ | класс/объект, реализующий interface поверх SDK |

## Почему асимметрия

Интерфейс — инструмент **инверсии зависимости**. Инверсия нужна только там, где runtime-стрелка конфликтует с правилом «все импорты внутрь»:

- driving: роут и так вызывает и импортирует домен — конфликта нет, прослойка = церемония
- driven: домен вызывает PayPal в runtime, но импортировать SDK нельзя → интерфейс в домене переворачивает import-стрелку (`StripeGateway ──import──▶ PaymentGateway`)

## Driving-порт в коде

```typescript
// domain/cancel-order.ts — вот это ВСЁ ВМЕСТЕ = driving-порт:
export type CancelOrderCommand = { orderId: string; reason: CancelReason }
export type CancelOrderResult  = { refundAmount: number }
export async function cancelOrder(cmd: CancelOrderCommand, deps: Deps): Promise<CancelOrderResult>
```

Контракт проверяет компилятор: адаптер передал не тот `cmd` — не собралось. Второй адаптер (CLI, cron, тест) зовёт ту же функцию: порт один, адаптеров много.

## Когда явный interface на driving всё же пишут

- DI-контейнер требует интерфейс для связывания (Java/NestJS: `interface CancelOrderUseCase` + класс)
- декораторы вокруг use case: логирование / транзакция / авторизация оборачивают реализацию
- контракт фиксируют раньше реализации (разделённые команды)

В TS/функциональном стиле — почти всегда лишнее: экспортированная сигнатура уже контракт.

## Мнемоника

**Интерфейс — только против стрелки.** Вызов внутрь → просто зови функцию. Вызов наружу → только через объявленный в домене interface.
