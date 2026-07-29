---
title: GraphQL — как работает и зачем нужен
tags: [concept, api, graphql, networking]
date: 2026-04-09
---

# GraphQL — как работает и зачем нужен

next: [[rest-api]] [[grpc]] [[api-gateway]] [[edge-vs-serverless]]

---

```
Client query { user { name, posts } }
        │
        ▼
  POST /graphql
        │
   [Schema + Resolvers]
        ├──▶ resolver: user    ──▶ DB (users)
        └──▶ resolver: posts   ──▶ DB (posts)
        │
        ▼
  { data: { user: { name, posts } } }
```

## Что это

GraphQL — язык запросов к API, где **клиент сам описывает**, какие данные ему нужны. Создан Facebook в 2012, открыт в 2015.

В отличие от REST, где сервер решает что отдать (фиксированные эндпоинты), в GraphQL клиент формирует запрос — и получает ровно то, что попросил.

## Как работает

### Один эндпоинт

REST: десятки URL (`/users`, `/users/1/posts`, `/posts/1/comments`).
GraphQL: один `POST /graphql`, всё через тело запроса.

### Запрос (Query)

Клиент описывает структуру нужных данных:

```graphql
query {
  user(id: "42") {
    name
    email
    posts(last: 3) {
      title
      commentCount
    }
  }
}
```

Ответ повторяет структуру запроса:

```json
{
  "data": {
    "user": {
      "name": "Tony",
      "email": "tony@example.com",
      "posts": [
        { "title": "First post", "commentCount": 5 },
        { "title": "Second post", "commentCount": 2 },
        { "title": "Third post", "commentCount": 0 }
      ]
    }
  }
}
```

В REST для этого нужно 2-3 запроса: `/users/42` + `/users/42/posts` + для каждого поста — счётчик комментариев.

### Схема (Schema)

На сервере описывается **схема** — контракт между клиентом и сервером:

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts(last: Int): [Post!]!
}

type Post {
  id: ID!
  title: String!
  commentCount: Int!
  author: User!
}

type Query {
  user(id: ID!): User
}
```

- `!` — поле обязательное (non-null)
- Схема типизирована — клиент знает заранее, что можно запросить
- Это и документация, и валидация в одном

### Resolver'ы

Каждое поле в схеме привязано к **resolver'у** — функции, которая знает, откуда взять данные:

```
Query.user     → достать из БД users
User.posts     → достать из БД posts WHERE author_id = user.id
Post.author    → достать из БД users WHERE id = post.author_id
```

Resolver'ы вызываются **только для запрошенных полей**. Не попросил `posts` — resolver не сработает.

### Мутации (Mutations)

Для изменения данных — отдельный тип `Mutation`:

```graphql
mutation {
  createPost(input: { title: "New post", body: "..." }) {
    id
    title
  }
}
```

Мутация возвращает результат — можно сразу получить созданный объект без дополнительного запроса.

### Подписки (Subscriptions)

Реалтайм через WebSocket:

```graphql
subscription {
  newComment(postId: "1") {
    text
    author { name }
  }
}
```

Сервер пушит данные клиенту при появлении нового комментария.

## Ключевые свойства

| Свойство | Что значит |
|---|---|
| **Нет overfetching** | Получаешь только запрошенные поля, не весь объект |
| **Нет underfetching** | Один запрос вместо цепочки REST-вызовов |
| **Типизация** | Схема = контракт + документация + валидация |
| **Интроспекция** | Клиент может запросить саму схему (`__schema`) — автогенерация типов, автодополнение в IDE |

## Где применяют и почему

### GitHub CLI (`gh`)

GitHub перешёл на GraphQL (API v4) потому что их граф сущностей глубокий: repo → PR → reviews → comments → users. Одна команда `gh pr view` запрашивает данные из 5+ сущностей — в REST это 5 запросов с rate limit, в GraphQL — один.

### PayPal (валидация транзакций)

GraphQL как **API Gateway** поверх микросервисов. Один запрос на валидацию, а resolver'ы параллельно дёргают account, fraud-scoring, balance, compliance:

```
Клиент → GraphQL Gateway → ┬─ Account Service
                            ├─ Fraud Service
                            ├─ Balance Service
                            └─ Compliance Service
```

Без этого фронтенд сам оркестрирует 4 вызова и склеивает ответы.

### Общий паттерн

GraphQL выигрывает когда:
- **Граф связанных сущностей** — много вложенных связей (соцсети, маркетплейсы)
- **Разные клиенты** — web, mobile, CLI запрашивают разные наборы полей
- **API Gateway** — единая точка входа в микросервисы
- **Публичное API** — rate limit на запрос, а не на поле

## Когда GraphQL — лишнее

- Простой CRUD с 1-2 сущностями — REST проще
- Service-to-service — gRPC эффективнее (бинарный протокол, стриминг)
- File upload / streaming — GraphQL не для этого
- Кэширование — REST кэшируется тривиально (по URL), в GraphQL нужен отдельный слой (Apollo Cache, Relay)
