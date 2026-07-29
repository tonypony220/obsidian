---
title: CDN Caching
tags: [concept, networking]
date: 2026-04-12
---

# CDN Caching

moc: [[networking-moc]]
next: [[edge-vs-cdn]]

---

```
Browser ──▶ Edge PoP ──[HIT]──▶ cached response
                  │
               [MISS]
                  │
                  ▼
              Origin Server ──▶ Edge PoP (store) ──▶ Browser
```

Кэширование контента на PoP (Point of Presence) — серверах, географически близких к пользователю.

## Как работает

Первый запрос → CDN проксирует к origin → сохраняет ответ → все последующие запросы отдаёт из кэша. Cache key обычно = URL + параметры.

## Что кэшируется

- Статика: JS, CSS, шрифты, картинки
- Медиа: видео, PDF, аудио
- API-ответы с `Cache-Control` заголовками (реже)

## Cache invalidation

Главная сложность CDN-кэширования. Основные стратегии:
- **TTL** — время жизни: `Cache-Control: max-age=3600`
- **Purge** — ручная инвалидация конкретного URL или по тегу
- **Stale-while-revalidate** — отдать старое, обновить в фоне

## Cache key

Каждая комбинация параметров = отдельный cache entry. Пример для картинок: `image.jpg?w=800&q=75` и `image.jpg?w=400&q=90` — два разных кэша.
