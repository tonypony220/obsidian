---
title: Service Worker — programmable network proxy in browser
tags: [concept, browser, frontend, networking]
date: 2026-05-13
---

# Service Worker

moc: [[front-moc]]
next:
- [[sw-scope]]
- [[sw-lifecycle]]
- [[sw-fetch-interception]]
- [[cdn-caching]]

---

```
                ┌─── scope: /app/* ───────────────┐
                │                                 │
   [page]  ─fetch()─▶  ┌──────────────┐  ─fetch()─▶  [network]
                       │ Service      │
   [page]  ◀response── │ Worker       │ ◀response── [network]
                       │              │
                       │  fetch       │   ──▶  [Cache Storage]
                       │  push        │
                       │  sync        │
                       └──────────────┘
                │                                 │
                └─────────────────────────────────┘
              вне scope → SW не виден, запрос идёт напрямую
```

**TL;DR:** SW — скрипт-прокси внутри браузера между страницей и сетью. Перехватывает запросы только в пределах своего `scope` и только если зарегистрирован `fetch` event listener — без listener'а SW существует, но network-прозрачен. Главная мотивация — offline-first (ответить из локального кеша вместо сети); push, background sync, PWA install — побочные продукты той же архитектуры.

## Что это

JS-скрипт, который браузер запускает в отдельном глобальном контексте (без DOM, без `window`), параллельно странице. Регистрируется через `navigator.serviceWorker.register('/sw.js')` и после активации сидит между страницей и network-стеком.

Главное свойство: может отвечать на запросы вместо реальной сети. Это и есть «программируемый прокси».

## Зачем перехватывать вообще

Изначальная мотивация — **offline**. До SW был AppCache (плохо спроектированный), SW — его наследник.

Сценарий: открыто веб-приложение, интернет пропал. Без SW — каждый `fetch()` падает с network error. С SW — он ловит fetch и достаёт ответ из заранее набитого [[cdn-caching|кеша]] (`Cache Storage` API, отдельный от HTTP-кеша).

Всё остальное — побочные продукты этой архитектуры:

| Use-case | Что использует |
|---|---|
| Offline-first PWA | fetch handler + Cache Storage |
| Performance кеш (stale-while-revalidate) | fetch handler + cache strategy |
| Push-нотификации | `push` event (без fetch перехвата вообще) |
| Background sync | `sync` event — досылка после возврата сети |
| Periodic sync | браузер сам будит SW раз в N часов |
| Install-prompt (Add to Home Screen) | сам факт регистрации SW + manifest |

## Handler-driven — opt-in, не auto

Регистрация SW **не** означает автоматический перехват. Перехватывается только то, на что есть зарегистрированный listener:

```js
// этот SW НЕ перехватывает fetch'и:
self.addEventListener('push', handlePush);
self.addEventListener('notificationclick', handleClick);
// ← никакого 'fetch' listener'а → network-прозрачен

// этот ловит всё в scope:
self.addEventListener('fetch', (event) => {
  event.respondWith(cacheFirst(event.request));
});

// а этот — выборочно:
self.addEventListener('fetch', (event) => {
  if (new URL(event.request.url).pathname.startsWith('/api/')) return;
  event.respondWith(cacheFirst(event.request));
});
```

Если `respondWith()` не вызван — браузер делает запрос напрямую. Если вызван — ответом для страницы становится то, что вернула переданная промис-функция.

## Что НЕ перехватывается

В пределах [[sw-scope|scope]] SW ловит navigation, subresources и cross-origin fetch'и инициированные контролируемыми страницами. Но **не** ловит:

- свой собственный JS-файл при обновлении (иначе SW не мог бы сам себя заменить)
- WebSocket frames после handshake — после установки соединения SW из цепочки выпадает
- запросы внутри dedicated/shared workers — у воркеров своя fetch-стека
- cross-origin запросы из других вкладок — каждый SW обслуживает только свой origin
- запросы инициированные браузером, не страницей (`<link rel="prefetch">` — зависит от браузера)

## Концепция шире

Программируемый прокси-слой между клиентом и сетью — общий паттерн:

- **Nginx / Envoy / Traefik** — reverse proxy перед приложением: кешируют, переписывают, маршрутизируют. SW — то же самое, только локально в браузере и для одного origin.
- **iptables / netfilter** — программируемый перехват пакетов на уровне ядра Linux. Хуки на стадиях (PREROUTING, FORWARD, …) — аналог `fetch`/`push`/`sync` events.
- **HTTP middleware** (Express, Koa, Fastify) — цепочка функций между request и handler; каждая может ответить сама или пропустить дальше.
- **eBPF** — программируемые хуки в ядре, перехватывают syscalls, packets, tracepoints. Та же модель: declarative subscribe → callback → решение пропустить или подменить.

Общее везде: **точка вставки в готовый pipeline + opt-in регистрация listener'ов + возможность ответить вместо нижележащего слоя или пропустить дальше**.
