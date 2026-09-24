---
title: Edge vs CDN
tags: [concept, tradeoff]
date: 2026-04-09
---

# Edge vs CDN

moc: [[networking-moc]]
next: [[cdn-caching]]

---

```
CDN:   Browser ──▶ Edge PoP ──▶ [cached file] ──▶ Browser
                       │ miss
                       ▼
                   Origin

Edge:  Browser ──▶ Edge PoP ──▶ [run code] ──▶ Browser
                   (compute)
```

## Коротко

**CDN** — сеть серверов по миру, которая кэширует и отдаёт статику. Только отдача, никакой логики.

**Edge** — те же географически распределённые серверы, но с возможностью выполнять код. По сути CDN + compute.

> Каждый edge-сервер может быть CDN-нодой, но не каждый CDN умеет выполнять код.

## Как работает

```
CDN:    Браузер → Edge PoP → отдаёт закэшированный файл
Edge:   Браузер → Edge PoP → выполняет функцию → отдаёт результат
```

## CDN — Content Delivery Network

Задача — минимизировать латентность отдачи статического контента (картинки, JS, CSS, шрифты). Контент кэшируется на PoP (Point of Presence) — серверах, географически близких к пользователю.

Примеры:
- **Cloudflare CDN** — кэширует файлы из Supabase Storage, каждая комбинация параметров (width, quality) = отдельный cache key
- **CloudFront** (AWS) — CDN для S3, API Gateway и т.д.
- **Fastly**, **Akamai** — enterprise CDN

## Edge Compute

Задача — выполнять код максимально близко к пользователю, без cold start (или с минимальным). Обычно ограниченный runtime (нет полного Node.js).

Примеры:
- **Cloudflare Workers** — V8 isolates, код на тех же PoP что и CDN (~300 локаций)
- **Vercel Edge Functions** — Next.js middleware, edge runtime (`export const runtime = 'edge'`)
- **Supabase Edge Functions** — Deno на Fly.io (меньше точек чем Cloudflare)
- **Deno Deploy** — глобально распределённый Deno runtime

## Edge vs Serverless

|                  | Edge                          | Serverless (Lambda и т.д.)        |
|------------------|-------------------------------|-----------------------------------|
| Расположение     | 200+ PoP по миру              | Несколько регионов                |
| Cold start       | ~0ms (V8 isolates)            | 100ms–несколько секунд            |
| Runtime          | Ограниченный (Web APIs)       | Полный (Node.js, Python и т.д.)  |
| Ресурсы          | CPU/memory лимиты жёсткие     | Более гибкие лимиты              |
| Use case         | Routing, auth, A/B, i18n      | Тяжёлая логика, DB queries       |

## Практический пример: обработка картинок

```
Supabase Render API (CDN + edge compute):
Браузер → Cloudflare Edge → imgproxy (ресайз на первый запрос) → Storage
                          → кэш (все последующие запросы)

Next.js <Image> (serverless):
Браузер → Next.js сервер (/_next/image) → fetch оригинал → sharp resize → отдать
```

Первый вариант — ресайз на edge, кэш на CDN, нагрузка на инфраструктуру провайдера.
Второй — ресайз на твоём сервере, кэш локальный, нагрузка на твой Vercel.

## Когда что выбирать

**CDN** — статика, медиа, assets. Всё что можно закэшировать и отдать без логики.

**Edge** — лёгкая логика близко к юзеру: routing, redirects, auth checks, A/B тесты, геолокация, персонализация headers.

**Serverless** — тяжёлая бизнес-логика, работа с БД, длительные вычисления.
