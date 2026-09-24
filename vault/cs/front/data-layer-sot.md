---
title: Data layer SOT — где живёт state приложения
tags: [concept, pattern, react, frontend, architecture]
date: 2026-05-07
---

# Data layer SOT

moc: [[front-moc]]
back: [[design-tokens-sot]]
next:
- [[tanstack-query]]
- [[url-as-state]]
- [[zustand-store]]

---

```
  Источники state в приложении (от внешнего к локальному):

  ┌──────────────────────────────────────────────────────────────┐
  │  Server          ← business data (users, orders, messages)   │
  │                    клиент только кэширует                    │
  ├──────────────────────────────────────────────────────────────┤
  │  URL             ← shareable state (filters, page, modal-id) │
  │                    видно в адресной строке                   │
  ├──────────────────────────────────────────────────────────────┤
  │  localStorage    ← persistent client (theme, dismissed flags)│
  ├──────────────────────────────────────────────────────────────┤
  │  Global memory   ← shared UI state (Zustand, Context)        │
  │                    редко, точечно                            │
  ├──────────────────────────────────────────────────────────────┤
  │  Component state ← ephemeral UI (hover, focus, open, input)  │
  │                    useState, useReducer                      │
  └──────────────────────────────────────────────────────────────┘
```

**TL;DR:** Каждый кусок данных должен иметь **один SOT**, дублирование = sync-баги. Главная дихотомия — server state (чужие данные на сервере, нужен кэш+инвалидация → TanStack Query) vs client state (твои данные, useState/Zustand). Shareable state (фильтры, открытая модалка по ссылке) живёт в URL. Persistent (тема) — в localStorage. Form state — в react-hook-form.

## Принцип одного SOT

Один кусок данных — один источник истины. Если данные дублированы в двух местах → нужна синхронизация → race conditions, рассинхрон, useEffect-связки, баги.

```tsx
// ❌ Filter живёт и в URL, и в local state
const [filter, setFilter] = useState('');
const [params, setParams] = useSearchParams();

useEffect(() => {
  setFilter(params.get('filter') ?? '');  // sync URL → local
}, [params]);

const handleChange = (v: string) => {
  setFilter(v);
  setParams({ filter: v });               // sync local → URL
};
```

Два SOT, два направления синхронизации, useEffect-зависимости. Хрупко.

```tsx
// ✅ URL — единственный SOT
const [params, setParams] = useSearchParams();
const filter = params.get('filter') ?? '';
const setFilter = (v: string) => setParams({ filter: v });
```

Нет дубля, нет sync-логики, нет багов.

## Главное разделение: Server state vs Client state

Самая важная дихотомия современного фронтенда. Разные жизненные циклы → разные инструменты.

| | Server state | Client state |
|---|---|---|
| Кому принадлежит | серверу | тебе |
| Может ли измениться без твоего ведома | да | нет |
| Устаревает (stale)? | да | нет |
| Race conditions с сетью | да | нет |
| Нужен loading/error/refetch | да | нет |
| Инструмент | TanStack Query / SWR / Apollo | useState / useReducer / Zustand |

**Server state** — это users, orders, messages, posts. Не клади его в Redux/Zustand/Jotai — получишь 80% boilerplate'а на синхронизацию.

**Client state** — это `isOpen`, `selectedTab`, `inputValue`, `isCollapsed`. Тут TanStack Query избыточен.

## Историческая справка: Redux

До ~2019 стандартом был **Redux** — глобальный store с reducer'ом, в который клали **всё подряд**: и server data, и UI-флаги, и формы.

```
action {type, payload} → reducer (state, action) → new state → subscribers rerender
```

Боль:
- Огромный boilerplate (actions, action creators, reducers, selectors, middleware).
- Server data в Redux = руками писать `loading/success/error` для каждого endpoint'а.
- Async через `redux-thunk`/`redux-saga` — отдельная сложность.

После 2019 индустрия разделила задачи:
- Server state ушёл в **TanStack Query** (закрыл 80% боли).
- Локальный UI state — в `useState`.
- Shared client state — в **Zustand** / **Jotai** (легковесные альтернативы).
- Redux Toolkit (RTK) сделал API удобнее, но Redux всё реже выбирают для новых проектов.

Знать про Redux — полезно (увидишь в legacy). Учить как первое — нет.

## URL as state — почему критично

Если данные **shareable** или **bookmarkable** — их SOT обязан быть URL.

В URL:
- Фильтры списка (`?status=open&assignee=me`)
- Поиск (`?q=foo`), пагинация (`?page=3`), cursor (`?after=abc`)
- Текущая вкладка (`?tab=settings`)
- ID элемента в детальной view (`/orders/42`)
- Открытая модалка, если её нужно открывать по ссылке (`?modal=invite`)

Зачем:
- **Shareable** — кинул ссылку, коллега видит то же.
- **Browser back/forward** работает естественно.
- **Refresh не теряет state**.
- **SSR-friendly** — сервер видит URL и сразу рендерит нужное.

НЕ в URL:
- Hover, focus, scroll position
- Содержимое поля до submit
- Ephemeral toasts, открыт ли dropdown

## useState vs Zustand — когда что

`useState` хорош для **локального** state. Когда тот же state нужен в нескольких разнесённых местах дерева — начинается **prop drilling**.

### Пример проблемы

Корзина нужна в `<Header>`, в `<Product>` (глубоко в каталоге) и в `<CartPage>`:

```tsx
function App() {
  const [cart, setCart] = useState<Item[]>([]);
  return (
    <>
      <Header cart={cart} />
      <Main>
        <CategoryList cart={cart} setCart={setCart}>
          <Category cart={cart} setCart={setCart}>
            <Product cart={cart} setCart={setCart} item={item} />
          </Category>
        </CategoryList>
      </Main>
      <CartPage cart={cart} setCart={setCart} />
    </>
  );
}
```

Промежуточные `Main`, `CategoryList`, `Category` получают `cart`/`setCart` только для прокидывания вниз. Им самим эти данные не нужны. Это и есть prop drilling.

### Решение: Zustand

```tsx
// store.ts
import { create } from 'zustand';

const useCart = create<{
  items: Item[];
  add: (item: Item) => void;
}>(set => ({
  items: [],
  add: (item) => set(s => ({ items: [...s.items, item] })),
}));

// Header.tsx
const count = useCart(s => s.items.length);   // подписка на длину
// Product.tsx
const add = useCart(s => s.add);              // подписка на функцию
// CartPage.tsx
const items = useCart(s => s.items);
```

Никакого prop drilling. Каждый компонент читает только то, что нужно. Промежуточные слои ничего про корзину не знают.

### Zustand vs Context

Context тоже решает prop drilling, но **ререндерит всех** consumer'ов при любом изменении value.

Zustand даёт **точечные подписки**: компонент подписался на `items.length` → ререндерится только при изменении длины. Изменился порядок элементов с той же длиной — компонент не дёрнется.

Правило:
- **Context** — для редко меняющихся broadcast-данных: текущий юзер, тема, локаль.
- **Zustand/Jotai** — для часто меняющихся shared-данных.

### Когда useState хватает

- State в одном компоненте → useState.
- State в прямых детях (1-2 уровня) → лифтнул до общего предка, передал prop'ом.

### Когда нужен Zustand

- State в 3+ местах, разнесённых по дереву.
- Lifting вызывает prop drilling через 4+ слоёв.
- Часто меняется (Context дал бы массовые ререндеры).

В большинстве проектов Zustand нужен совсем чуть-чуть: тема, current user, корзина, открытые глобальные модалки.

## Form state — отдельная история

Формы — пограничный случай. Ввод часто меняется (controlled = ререндер на keystroke), нужны валидация и dirty-tracking, но **локально** для одного экрана.

- **react-hook-form** — точечные подписки через ref'ы, минимум ререндеров. Стандарт 2024+.
- **Formik** — был лидером, уступил по перформансу.
- **Native FormData** — для простых форм без валидации можно вообще без библиотеки.

Не клади form state ни в Redux, ни в Zustand, ни в URL.

## Persistent state — localStorage

Что должно пережить refresh / новую сессию:
- Тема (light/dark)
- Свернут ли sidebar
- Dismissed-флаги (баннеры, тосты «больше не показывать»)
- Недавние поиски, последний выбранный фильтр (если не в URL)

Простейший хук — `useLocalStorage(key, default)` (есть в `usehooks-ts`, `@uidotdev/usehooks` и т. п.).

Не клади туда:
- Аутентификационные токены — это XSS-риск, использовать httpOnly-cookies.
- Большие данные — есть лимит ~5 MB, для крупных кэшей IndexedDB.

## Правила-эвристики

1. **Server data → TanStack Query.** Никогда не Redux/Zustand для server state.
2. **Shareable / bookmarkable → URL.** Фильтры, пагинация, открытая модалка.
3. **Persistent across sessions → localStorage.** Тема, dismissed-флаги.
4. **Ephemeral UI → useState.** Hover, focus, open-флаги.
5. **Shared UI state → лифтить useState до общего предка; в Zustand если глубоко.**
6. **Form state → react-hook-form.** Не глобальный store.
7. **Один кусок данных = один SOT.** Дублирование = sync-баги.

## Распределение по правилу 80/20

В типичном современном приложении:

| Тип | Доля | Инструмент |
|---|---|---|
| Server state | ~80% | TanStack Query |
| Local UI state | ~15% | useState / useReducer |
| URL state | ~4% | router params, useSearchParams |
| Global client state | ~1% | Zustand / Jotai |
| Persistent | <1% | localStorage |

Если в проекте Redux/Zustand занимает 50% data — это знак, что туда попал server state, который должен быть в Query.

## Концепция шире

Принцип «один SOT, остальное derived» встречается везде, где есть синхронизация:

- **БД нормализация (3NF)** — каждый факт хранится в одном месте, остальное — внешние ключи / VIEW. Денормализация = ручная синхронизация.
- **Git** — коммиты как SOT, ветки/теги — ссылки. Не дублируется код, дублируются указатели.
- **DNS** — authoritative server держит истину, caching resolvers — копии с TTL и инвалидацией.
- **CRDT / Event Sourcing** — events как SOT, state — derived свёрткой. Снимки — кэш, не истина.
- **CPU caches (L1/L2/L3)** — main memory SOT, кэши — копии с инвалидацией по cache coherence протоколу (MESI).
- **Документация vs код** — код SOT поведения, доки — производное (потому в CLAUDE.md правило «устаревшие доки хуже их отсутствия»).

Общее везде: **дублирование данных без явного SOT и протокола инвалидации = неизбежная рассинхронизация**. Один источник истины + правила derive — единственный устойчивый паттерн.
