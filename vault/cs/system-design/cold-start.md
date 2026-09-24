---
title: Cold Start
tags: [concept, tradeoff]
date: 2026-04-12
---

# Cold Start

next: [[edge-vs-serverless]] [[edge-vs-cdn]]

---

```
Cold:  request ──▶ [load code] ──▶ [init runtime] ──▶ [run] ──▶ response
                    └────────────── 100ms–5s ────────────────┘

Warm:  request ──▶ [run] ──▶ response
                    └─ <5ms ─┘
```

Задержка при первом вызове serverless-функции, пока runtime инициализируется.

## Почему возникает

Serverless-платформа создаёт контейнер/изолят по запросу. Первый вызов включает: загрузку кода → инициализацию runtime → выполнение. Последующие вызовы переиспользуют "тёплый" контейнер.

## Порядок величин

| Платформа | Cold start |
|---|---|
| Cloudflare Workers (V8 isolates) | ~0-1ms |
| AWS Lambda (Node.js) | 100-500ms |
| AWS Lambda (Java) | 1-5s |
| Google Cloud Functions | 100ms-2s |

## Как уменьшить

- **Provisioned concurrency** — держать N тёплых инстансов (AWS Lambda)
- **Edge runtime** — V8 isolates вместо контейнеров (~0ms cold start)
- **Лёгкий runtime** — Node.js стартует быстрее Java
- **Меньше зависимостей** — меньше кода = быстрее загрузка
