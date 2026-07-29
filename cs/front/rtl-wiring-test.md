---
title: RTL Wiring Test
tags: [concept, pattern, testing, react, frontend]
date: 2026-07-04
---

# RTL Wiring Test

moc: [[test-strategy-moc]]
back: [[component-layers]]
next:
- [[coverage-theatre]]
- [[walking-skeleton]]
- [[round-trip-test]]
- [[functional-core-imperative-shell]]

---

```
       user event
           │
           ▼
   ┌───────────────┐
   │  <Component>  │  ← RTL: click / type
   └───────┬───────┘
           │  handler({ ...state })
           ▼
   ┌───────────────┐
   │  hook / lib   │  ← реально исполняется
   │   (builder)   │
   └───────┬───────┘
           │  fetch(url, body)
           ▼
     ═══ NETWORK ═══   ← MSW перехватывает
                        assert: { url, method, body }
```

**TL;DR:** Runtime-пин связки «click → нужный builder вызван с правильными X». Мокается **сеть** (MSW/fetch), НЕ сам builder — иначе тест сверяет своё ожидание со своим же кодом ([[coverage-theatre]]). Ловит хардкод вместо state'а, вызов не того builder'а, опечатки в именах полей, disconnected кнопку.

## Что такое RTL

React Testing Library — библиотека тестирования React-компонентов «как видит пользователь»: искать элементы по видимой роли/тексту, кликать, вводить, ждать реакции. Не лезть в state/props/internal — это ломается на каждом рефакторе.

Стек в целом:
- **Vitest / Jest** — раннер, `expect`, `vi.mock`
- **jsdom** — эмулятор браузерного DOM в Node.js
- **RTL** — `render`, `screen.getByRole(...)`, `fireEvent`, `waitFor`
- **MSW** (Mock Service Worker) — перехватчик `fetch`/XHR; в браузере через service worker, в тестах через Node.js hook

## Wiring test — суть

Тестирует стык «тонкая обёртка (Container/hook) → domain-слой». Проверяет одну вещь: **click правильной кнопки вызвал правильный builder с правильными аргументами**.

TypeScript ловит часть этих багов (сигнатура builder'а), но не всё:
- хардкод вместо переменной (`qty: 1` вместо `qty: state`) — TS не видит
- два builder'а с похожими сигнатурами (`performAdd` vs `performRemove`) — TS не видит какой ты выбрал
- кнопка без `onClick` — TS молчит
- опечатка в имени поля когда обе стороны свободно типизируются — TS не всегда ловит

Дефект живёт в **декоративном слое**, между JSX и вызовом builder'а. Именно там его и надо ловить.

## Что именно ломается

### 1. Хардкод вместо state'а

```tsx
const [qty, setQty] = useState(1)

<input value={qty} onChange={e => setQty(+e.target.value)} />
<button onClick={() => performAdd({ listingId, qty: 1 })}>Add</button>
                                              // ↑ забыли заменить на qty
```

RTL: `type("3") → click → MSW body: { qty: 1 }`. Ожидали `3`. FAIL.

### 2. Не тот builder

```tsx
<button onClick={() => performRemove({ listingId })}>Add</button>
                    // ↑ copy-paste из соседнего компонента
```

RTL: `click → MSW method: DELETE` (ожидали POST). FAIL.

### 3. Опечатка в имени поля

```tsx
<button onClick={() => performAdd({ id: listingId, qty })}>Add</button>
                                //  ↑ builder ждёт listingId
```

Builder делает `JSON.stringify({ listingId: undefined, qty: 1 })` — `listingId` не был передан → `undefined` → `JSON.stringify` его выкидывает → на wire уходит `{"qty":1}`. RTL: MSW видит body без `listingId`. FAIL.

### 4. Кнопка без onClick

```tsx
<button>Add to Cart</button>   // забыли onClick при рефакторе
```

RTL: `click → MSW: (no requests)`. FAIL.

## Центральная дисциплина: мокать сеть, НЕ builder

Это главная развилка теста. Легко нарушить, легко превратить в [[coverage-theatre]].

### ❌ Мок builder'а

```ts
vi.mock('~/lib/cart/add-to-cart')                                          // (1)
render(<AddToCart listingId="abc" />)                                      // (2)
click('Add')                                                               // (3)
expect(mockPerformAdd).toHaveBeenCalledWith({ listingId: 'abc', qty: 1 })  // (4)
```

**(1)** `vi.mock(...)` заменяет весь модуль на автомок **до** его импорта в компоненте. Функция `performAdd` становится fake-функцией, которая ничего не делает, но запоминает вызовы.

**(2)** `render` монтирует компонент в jsdom-DOM. Компонент импортит `performAdd` — получает mock.

**(3)** `click('Add')` — упрощение для `fireEvent.click(screen.getByRole('button', { name: 'Add' }))`. Внутри `onClick` вызывается `performAdd(...)` — но это mock, он просто записывает: «меня позвали с этим».

**(4)** `expect(mockPerformAdd).toHaveBeenCalledWith(...)` — сверяем что mock был позван с указанным объектом.

**Почему это тавтология.** В компоненте: `performAdd({ listingId, qty: 1 })`. В `expect`: то же самое `{ listingId, qty: 1 }`. Мы сравниваем свой код со своим ожиданием, минуя реальную функцию. Если и в компоненте, и в `expect` опечатка `id` вместо `listingId` — оба зелёные, баг летит на прод.

Ничего реально не запустилось: ни настоящий `performAdd`, ни `fetch`, ни сериализация. Тест защищает только «клик кнопки вызвал функцию с ожидаемым нами объектом».

### ✅ Мок сети (MSW)

```ts
render(<AddToCart listingId="abc" />)              // (1)
click('Add')                                       // (2)
await waitFor(() => {                              // (3)
  expect(msw.lastRequest()).toMatchObject({        // (4)
    url: '/api/cart',
    method: 'POST',
    body: { listingId: 'abc', qty: 1 },
  })
})
```

**(1)** То же самое, но **без** `vi.mock`. Компонент импортит **настоящий** `performAdd`.

**(2)** Симулируем клик. `onClick` реально вызывает `performAdd({ listingId, qty })`. Внутри:
```ts
fetch('/api/cart', {
  method: 'POST',
  body: JSON.stringify({ listingId, qty }),
})
```

**(3)** `await waitFor(() => ...)` — асинхронное ожидание. Между кликом и появлением запроса в MSW есть промежуточные микротаски (promise chain внутри fetch). `waitFor` повторяет колбэк пока не пройдёт `expect` или не выйдет таймаут (default ~1 сек). Без `await` тест может упасть с «нет запросов» — просто не успел.

**(4)** MSW заранее настроен на перехват POST `/api/cart`:
```ts
const captured: Request[] = []
server.use(
  http.post('/api/cart', ({ request }) => {
    captured.push(request.clone())
    return HttpResponse.json({ ok: true })
  })
)
```
`msw.lastRequest()` в примере — упрощение для `captured.at(-1)`. `.toMatchObject({...})` — partial match, чтобы не заботиться о хедерах.

**Что реально проверилось (7 шагов цепочки):**
1. Компонент отрендерился без ошибок
2. Кнопка «Add» доступна и находится по роли
3. `onClick` подключён
4. `onClick` вызвал `performAdd`
5. `performAdd` дошёл до `fetch`
6. `fetch` идёт на правильный URL с правильным методом
7. Тело сериализуется в правильную форму

Мокается только **внешняя граница** — сеть. Всё между кнопкой и сетью реально исполняется.

## Что RTL wiring НЕ ловит

| Класс бага | Ловит? | Кем ловится |
|---|---|---|
| Клик не на ту кнопку / нет `onClick` | ✅ | RTL |
| Хардкод вместо state / переменной | ✅ | RTL |
| Не тот builder / метод | ✅ | RTL |
| Опечатка в имени поля | ✅ | RTL |
| Builder шлёт на неверный URL | ⚠ частично | RTL если URL в assert; иначе [[walking-skeleton]] |
| Schema не совпадает с canonical типом | ❌ | [[round-trip-test]] |
| Server-side логика | ❌ | integration/e2e |
| Ошибка сериализации внутри builder'а (кастомный transform) | ✅ | RTL (реальный builder + MSW видит результат) |
| Стили, a11y-контраст, отзывчивость | ❌ | visual regression / axe |
| Race condition при двойном клике | ❌ | стрессовый тест / отдельный тест на idempotency |

## Место на карте

Смотри [[test-strategy-moc]]. RTL wiring режет **верхнюю левую часть** пайплайна: `UI → hook → builder` с мок'ом на HTTP-границе. [[round-trip-test]] режет один слой (schema ↔ type) без UI и без HTTP. [[walking-skeleton]] режет всю цепочку насквозь.

Дополняют, не заменяют:
- RTL wiring отвечает «вызов дошёл до сети с правильным body»
- round-trip отвечает «schema и canonical type совпадают»
- walking skeleton отвечает «реальный сервер отвечает на реальный запрос из UI»

Каждый ловит свой класс бага. Убрать RTL — вернутся #1–#4 из «Что ломается».

## Не только React

Паттерн общий: «тонкая обёртка вызывает домен с правильными аргументами». Не React-специфика:

| Стек | Аналог |
|---|---|
| Vue | `@vue/test-utils` + `fireEvent` + MSW |
| Svelte | `@testing-library/svelte` + MSW |
| iOS UIKit / SwiftUI | UI-тест: tap button + assert `URLProtocol` перехватил запрос |
| Android Compose | composable test + `performClick` + retrofit interceptor |
| CLI | функциональный тест: запуск бинаря с argv + assert side effect на mock IO (fs, HTTP) |
| Backend HTTP handler | integration: request → assert domain function called with X (мокается DB, не domain) |

Общее везде: **мокается ближайшая внешняя граница за домен-слоем** — не сам домен. Если мокаешь домен — тестируешь свою же уверенность, а не поведение цепочки.
