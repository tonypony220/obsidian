---
title: Seam — тактический шов
tags: [architecture, pattern, refactoring, legacy, testing]
date: 2026-06-13
---

# Seam

back: [[architecture-stack]]
next:
- [[hex-architecture]]
- [[clean-architecture]]
- [[functional-core-imperative-shell]]
- [[module-organization-patterns]]

---

```
                 seam = подменяемая точка

    ┌────────┐       ┌──────────────┐
    │ caller │──────▶│  contract    │   ← caller не меняется
    └────────┘       └──────┬───────┘
                            │ swap здесь
                     ┌──────┴──────┐
                     ▼             ▼
               ┌──────────┐  ┌──────────┐
               │   prod   │  │   alt    │
               │ (fetch)  │  │ (mock,   │
               │          │  │  RN, …)  │
               └──────────┘  └──────────┘
```

**TL;DR:** seam — точка в коде, где поведение можно подменить, не правя caller'а. Это **тактика** (один шов под конкретную боль), не архитектура. Hex/Clean/Onion = система seam'ов в строго определённых местах + дисциплина их соблюдать.

## Определение

Майкл Фэзерс, *Working Effectively with Legacy Code* (2004): **a place where you can alter behavior in your program without editing in that place**. Шов — это контур, по которому код можно «разрезать» и подменить одну сторону, оставив другую нетронутой.

Тактика для legacy: внести тестируемость в кусок, который нельзя переписать целиком. Точка вариативности прокладывается там, где болит, и только там.

## Один пример

Было — caller намертво прибит к реализации:

```typescript
// pageA.tsx
async function loadUser(id: string) {
  const res = await fetch(`/api/users/${id}`)   // прямой fetch
  return res.json()
}
```

Прокладываем шов — выделяем контур, который caller дёргает по имени:

```typescript
// api/users.ts — seam
export async function getUser(id: string) {
  return http.get(`/users/${id}`)               // http — единственный знающий про fetch
}

// pageA.tsx
const user = await getUser(id)                  // caller не знает про fetch
```

Теперь подмена возможна:
- в тестах — `http.get` мокается одной строкой
- в React Native — `http` реализуется поверх RN-fetch, страница не правится
- завтра gRPC — переписан только `http`, callers молчат

## seam vs hex одной строкой

| | Seam | Hex |
|---|---|---|
| Что это | приём | архитектура |
| Масштаб | одна точка | вся система |
| Правило | «здесь можно подменить» | «domain не импортирует delivery» |
| Где | по диагностированным болям | по слоям, везде |
| Доказательство | подменили в тесте | `grep -r "from 'stripe'" domain/` пусто |

Hex = **система seam'ов в фиксированных местах (порты) + дисциплина их соблюдать.** Без дисциплины hex — это просто куча швов; без швов дисциплина hex не существует.

## Когда seam, а не hex

| Ситуация | Что брать |
|---|---|
| Legacy, нельзя переписывать целиком | seam в больные точки |
| Точечный порт на другую платформу (web → RN) | seam перед platform API |
| Один кусок надо тестировать, остальное и так чисто | seam перед грязью |
| Новый проект, домен богат, провайдеры меняются | hex с самого начала |
| `grep` должен доказать чистоту слоя | hex (правило, не приём) |

Анти-триггер для hex: «у нас три fetch'а в трёх местах и хочется протестировать» — это три seam'а, не Clean Architecture.

## Что конкретизирует seam

Seam — родовой термин. Конкретные техники реализации шва:

- **strategy / state** (GoF) — подменяемый алгоритм за интерфейсом
- **adapter / facade** (GoF) — переводчик/упроститель чужого API за нашим контуром
- **dependency injection** — передача реализации снаружи вместо `new` внутри
- **middleware / interceptor** — шов в виде цепочки обработчиков
- **plugin / hook point** — шов, объявленный заранее в host-системе

Все они отвечают на один вопрос Фэзерса: «как разрезать так, чтобы caller не знал?».

## Не только в коде

Тот же приём — точка отложенного решения о реализации — встречается там, где система должна допустить вариант без переделки целого:

- **#ifdef в C/C++** — preprocessor seam, ветка выбирается компиляцией
- **Dynamic linking (.so/.dll)** — link seam, реализация выбирается загрузчиком
- **Kernel modules / device drivers** — ядро объявило API, конкретный драйвер вставляется в runtime
- **Browser DevTools `Object.defineProperty`/extension content scripts** — runtime seam в чужой странице
- **Monkey-patching (Ruby/Python)** — seam, проложенный задним числом без согласия автора

**Общее везде:** caller обращается к стабильному имени, конкретная реализация выбирается *не в момент написания caller'а* (компиляцией / загрузкой / тестом / runtime'ом). Связь caller↔implementation сдвинута во времени — это и есть шов.
