---
title: Edge vs Serverless — когда edge реально полезен
tags: [edge, serverless, middleware, architecture]
date: 2026-04-09
---

# Edge vs Serverless — когда edge реально полезен

next: [[cold-start]] [[middleware-patterns]]

---

```
Edge (local work):
  User ──▶ Edge PoP ──▶ [logic in memory] ──▶ response   (~1ms)

Serverless (network calls):
  User ──▶ Lambda ──▶ fetch(Auth API) ──▶ fetch(DB) ──▶ response  (~300ms)
```

Для кейса с Supabase middleware (`getUser()`) — edge почти ничего не даёт, потому что каждый запрос всё равно летит к Supabase Auth API.

## Когда edge полезен

Edge выигрывает когда middleware делает **локальную работу без сетевых запросов**:

- **Rate limiting** — счётчик в памяти, проверка и 429, ноль сетевых вызовов
- **Редиректы** — `/old-url` → `/new-url`, чистая логика
- **Geo-routing** — показать другую страницу по стране из заголовка
- **A/B тесты** — выбрать вариант по cookie
- **Блокировка** — бот-детект, IP blacklist

Во всех этих случаях запрос обрабатывается за микросекунды рядом с пользователем и serverless function вообще не просыпается.

## Когда edge не помогает

Если middleware всё равно делает `fetch()` куда-то — разница только в cold start (~1ms edge vs ~100-500ms serverless). Плюс, но не принципиальный.

## Правило

> Чем больше логики можно выполнить **без сетевых вызовов** — тем больше смысла в edge.
