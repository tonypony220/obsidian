---
title: API Gateway
tags: [concept, pattern, architecture]
date: 2026-04-12
---

# API Gateway

moc: [[networking-moc]]
next: [[graphql]] [[grpc]] [[middleware-patterns]]

---

```
          ┌─────────────┐
Client ──▶│ API Gateway │──▶ User Service
          │             │──▶ Order Service
          │  auth/rate  │──▶ Payment Service
          └─────────────┘
```

Единая точка входа для клиентов в систему микросервисов. Клиент общается с одним URL, gateway маршрутизирует к нужному сервису.

## Что делает

- **Маршрутизация** — `/users` → User Service, `/orders` → Order Service
- **Аутентификация** — проверка токенов в одном месте
- **Rate limiting** — ограничение запросов по API key → алгоритмы: [[rate-limiter-algorithms]]
- **Агрегация** — один запрос клиента → несколько вызовов к сервисам → один ответ
- **Трансформация** — преобразование протоколов (REST → gRPC)

## Примеры

- AWS API Gateway
- Kong
- GraphQL как API Gateway (один эндпоинт, resolver'ы дёргают сервисы)

## Трейдофф

Плюс: клиенту проще, безопасность централизована, можно менять внутреннюю архитектуру.
Минус: single point of failure, дополнительная латентность, ещё один сервис для поддержки.
