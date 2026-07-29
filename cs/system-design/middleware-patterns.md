---
title: Middleware Patterns
tags: [pattern, architecture]
date: 2026-04-12
---

# Паттерны middleware

next: [[edge-vs-serverless]] [[higher-order-functions]]

---

```
Request
   │
   ▼
[Auth] ──next()──▶ [RateLimit] ──next()──▶ [Logging] ──next()──▶ [Handler]
   │                    │                      │                      │
   │◀───────────────────┴──────────────────────┴──────────────────────┘
   ▼
Response
```

Middleware — функция, которая перехватывает запрос/ответ в цепочке обработки. Ключевая абстракция: `(request, next) → response`.

## Chain of Responsibility

Каждый middleware решает: обработать и вернуть, или передать дальше через `next()`.

```
Request → Auth → RateLimit → Logging → Handler → Response
```

## Типичные задачи

- **Auth** — проверка токена, редирект на логин
- **Rate limiting** — счётчик запросов, 429 при превышении
- **Logging** — запись запроса/ответа
- **CORS** — установка заголовков
- **Redirect** — перенаправление старых URL

## Где живёт middleware

| Уровень | Пример |
|---|---|
| Edge | Cloudflare Workers, Vercel Edge Middleware |
| Web-фреймворк | Express, Next.js, Django |
| API Gateway | Kong, AWS API Gateway |

Правило: чем ближе к пользователю и чем меньше сетевых вызовов — тем лучше middleware на [[edge-vs-serverless|edge]].
