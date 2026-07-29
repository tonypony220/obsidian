---
title: gRPC
tags: [concept, api, networking]
date: 2026-04-12
---

# gRPC

moc: [[networking-moc]]
next: [[rest-api]] [[graphql]]

---

```
Client                          Server
  │  serialize (protobuf/binary)  │
  │ ──────── HTTP/2 frame ───────▶│
  │          [binary payload]     │ deserialize
  │                               │ handle RPC
  │ ◀──────── HTTP/2 frame ───────│
  │  deserialize (protobuf)       │
```

Remote Procedure Call фреймворк от Google. Бинарный протокол поверх HTTP/2.

## Ключевые свойства

- **Protocol Buffers** (protobuf) — бинарная сериализация, схема в `.proto` файлах
- **HTTP/2** — мультиплексирование, server push, сжатие заголовков
- **Кодогенерация** — из `.proto` генерируются клиент и сервер на любом языке
- **Стриминг** — unary, server-stream, client-stream, bidirectional

## Когда использовать

- **Service-to-service** — микросервисы общаются между собой (не через браузер)
- **Высокая нагрузка** — бинарный протокол эффективнее JSON
- **Стриминг** — реалтайм данные, логи, события
- **Полиглот** — сервисы на разных языках, схема одна

## Когда НЕ использовать

- Браузерные клиенты (нужен gRPC-Web прокси)
- Простой CRUD — REST проще
- Публичное API — REST/GraphQL понятнее для внешних разработчиков
