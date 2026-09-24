---
title: Option / Maybe
tags: [concept, pattern, type-system]
date: 2026-04-12
---

# Option / Maybe

moc: [[fp-patterns-moc]]
next:
- [[result-either]]
- [[algebraic-data-types]]
- [[total-vs-partial-functions]]
- [[parse-dont-validate]]

---

```
find_user(id)
      │
      ├── Some(user) ──► .map(u => u.name) ──► "Alice"
      │
      └── None ─────────────────────────────► "anonymous"
                                   unwrap_or()
```

Вместо `null` — тип "есть значение или нет". Заставляет обработать отсутствие явно.

```rust
// Rust — Option<T>
fn find_user(id: u32) -> Option<User> {
    // Some(user) или None
}

let name = find_user(42)
    .map(|u| u.name)        // трансформируем если есть
    .unwrap_or("anonymous"); // дефолт если нет
```

```swift
// Swift — Optional
let user: User? = findUser(42)
let name = user?.name ?? "anonymous"
```

```ts
// TypeScript — нет встроенного Option, но тот же принцип
function findUser(id: number): User | undefined {
    return users.find(u => u.id === id);
}
// TS заставит проверить: findUser(42).name — ошибка компиляции
```

## Где встречается

Rust `Option<T>`, Swift `Optional`, Java `Optional<T>`, Kotlin `?`, TypeScript strict null checks.

## Зачем

"Billion dollar mistake" — Тони Хоар (создатель null) сам назвал null своей главной ошибкой. Option делает отсутствие значения видимым в типе, а не скрытым сюрпризом в рантайме.

## Происхождение

ML (1973) → Rust, Swift, Kotlin (2010-е).
