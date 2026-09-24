---
title: Module Organization — Layered / Vertical Slice / Modular Monolith
tags: [architecture, pattern, modules, organization]
date: 2026-06-07
---

# Module Organization Patterns

back: [[hex-architecture]]
next:
- [[clean-architecture]]
- [[functional-core-imperative-shell]]

---

```
Layered (по слоям):           Vertical Slice (по фичам):     Modular Monolith (+ изоляция):

  controllers/                   features/checkout/             modules/checkout/
  services/                        ui logic api db                internal/ ← приватно
  repositories/                  features/orders/                 api.ts    ← единственный вход
  models/                          ui logic api db                events.ts
                                 features/payments/             modules/payments/
  поток сверху вниз                                              internal/
  кросс-фичные изменения =       1 фича = 1 папка                api.ts
  4 папки                        кросс-фичные связи              ESLint запрещает лезть
                                 через shared/                   в чужой internal/
```

**TL;DR:** Три способа **нарезать кодовую базу на куски** — по техническим слоям (Layered), по фичам (Vertical Slice), по фичам с жёсткими границами (Modular Monolith). Ортогонально [[hex-architecture]]/[[clean-architecture]]/[[functional-core-imperative-shell]]: те задают «что внутри слоя», эти — «как режем сам проект».

## 1. Layered (N-tier)

Папка верхнего уровня = технический слой. Поток сверху вниз.

```
src/
  controllers/  ← presentation
  services/     ← business
  repositories/ ← data
  models/
```

- **Поток:** `controller → service → repository → DB`
- **Откуда:** Java EE, Rails MVC, ASP.NET (90-е–2010-е)
- **Зависимости направлены вниз, к БД** — это **обратно от hex**, где домен ни от чего не зависит

**Проблемы:**
- Изменение одной фичи = охота по 4 папкам
- Сервисы переплетаются между собой (`CheckoutService → InventoryService → PaymentService`)
- Тест сервиса = моки трёх соседей

**Когда работает:** CRUD без сложной логики, маленький проект, одна команда.

## 2. Vertical Slice / Feature-based

Папка верхнего уровня = фича. Каждая фича — самодостаточный «пирог».

```
src/
  features/
    checkout/  { ui, logic, api, db }
    orders/    { ui, logic, api, db }
    payments/  { ui, logic, api, db }
  shared/      ← только реально кросс-фичное
```

- **Откуда:** Jimmy Bogard (~2017); естественно лёг на современные фреймворки
- **Next.js App Router / Remix / SvelteKit** — это vertical slice **по умолчанию** (`app/checkout/page.tsx` + `app/checkout/api/route.ts` рядом)

**Плюсы:**
- Изменение фичи = одна папка; удалить фичу = удалить папку
- Команды не сталкиваются в git
- Каждая фича может выбрать свою внутреннюю архитектуру (простая = inline, сложная = hex+FC/IS)

**Проблемы:**
- Куда деть общие типы — `shared/` пухнет, или одна фича зависит от другой
- Кросс-фичные сценарии (checkout трогает inventory + payments + orders) — где живёт координатор
- Соблазн дублирования утилит

**Когда работает:** средний и большой проект, несколько команд, фичи относительно независимы.

## 3. Modular Monolith

Vertical Slice + **жёсткие границы**. Модуль не может вызвать другой напрямую — только через явный публичный API.

```
src/
  modules/
    checkout/
      internal/    ← всё приватное
      api.ts       ← единственная публичная точка
      events.ts    ← что эмитит наружу
    payments/
      internal/
      api.ts
      events.ts
  shared-kernel/   ← минимум: Money, ISO date, errors
```

- **Откуда:** Шимон Шольнг (~2019); идея — bounded contexts из DDD
- **Enforcement:** ESLint `no-restricted-imports` на `**/internal/**`, TypeScript project references, dependency-cruiser/ts-arch

**Общение между модулями:**

```typescript
import { paymentsApi } from '@/modules/payments/api'        // ✅
// import x from '@/modules/payments/internal/db/foo'       // ❌ ESLint
```

Или через **события**: модуль эмитит `OrderPlaced`, другие подписываются. Без прямых вызовов.

**Плюсы:**
- **Готовность к микросервисам**: вынести модуль = заменить in-process вызовы на HTTP. Граница уже есть
- Команды реально независимы (линтер не даст случайно влезть)
- Bounded contexts: `Order` в `orders/` и в `analytics/` могут быть **разными типами** — это фича, не дублирование
- Изоляция инцидентов: баг в `inventory/internal/` не утечёт

**Проблемы:**
- Дороже на старте
- Дисциплина: одна ESLint-disable «ну тут быстрее напрямую» — границы потекли
- Транзакции через модули = saga / outbox, не SQL transaction

**Когда работает:** большой проект, 3+ команды, хотим готовности к микросервисам без оплаты сетевого оверхеда **сейчас**.

## Сводная таблица

| Параметр | Layered | Vertical Slice | Modular Monolith |
|---|---|---|---|
| Папка верхнего уровня | технический слой | фича | bounded context |
| Кросс-фичное изменение | 4 папки | 1 папка | 1 папка |
| Граница между фичами | нет | мягкая (соглашение) | **жёсткая** (линтер/типы) |
| Кросс-модульный вызов | прямой `import` | прямой `import` | только `api.ts` / events |
| Подходит для | CRUD, маленький | средний/большой | очень большой, готовность к μs |
| Современная норма для | Rails | React/Next/Remix | NestJS modules, Spring modulith |

## Ортогонально hex/Clean/FC/IS

Это **другая ось**. В любой организации можно применить любой уровень commitment'а:

| Организация | + Hex | + Clean | + FC/IS |
|---|---|---|---|
| **Layered** | `services/` импортируют интерфейсы | редко | внутри service-функции |
| **Vertical Slice** | каждая фича делает hex у себя | сложные фичи получают use cases | расчётные функции `logic/` чистые |
| **Modular Monolith** | каждый модуль внутри = hex | сложный модуль = Clean | вычислительное ядро = FC/IS |

**Внутри** модуля — степень commitment (hex / Clean / FC/IS). **Снаружи** — как модули нарезаны (Layered / Vertical / Modular).

## Эволюционная траектория

Чаще это **этапы роста**, а не выбор раз и навсегда:

```
MVP (1 чел, неделя)
   → один файл / Layered
      → 5–10 фич / Vertical Slice
         → 3 команды, границы плывут / Modular Monolith
            → нужен независимый deploy / Microservices
```

Каждый шаг — реакция на боль:
- «лезу в 4 папки на одну фичу» → **Vertical Slice**
- «случайно сломал чужую фичу» → **Modular Monolith**
- «деплоим всё ради одной строки» → **Microservices**

Не прыгать через ступени — overhead не окупится.

## Не только в коде

Тот же принцип «горизонтально vs вертикально vs изолированно» встречается везде, где надо нарезать сложную систему:

- **Книга:** по главам (vertical) vs по типам контента — список фигур, индекс, глоссарий (layered)
- **Завод:** конвейер по операциям (layered) vs ячейки полной сборки (vertical) vs отдельные цеха-bounded-contexts с интерфейсом «приёмка-отгрузка» (modular)
- **Город:** функциональные зоны (промзона / спальник / центр) — layered; mixed-use кварталы — vertical
- **Компания:** функциональные отделы (HR / финансы / разработка) — layered; продуктовые команды — vertical; автономные business units — modular

**Общее везде:** layered оптимизирует под глубокую специализацию слоя, vertical — под скорость cross-функциональных изменений, modular — под независимость единиц ценой накладных расходов на границы.
