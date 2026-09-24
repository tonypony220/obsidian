---
title: Result / Either
tags: [concept, pattern, type-system]
date: 2026-04-12
---

# Result / Either

moc: [[fp-patterns-moc]]
next:
- [[error-contract-union-vs-throw]]
- [[option-maybe]]
- [[interface-vs-discriminated-union]]
- [[parse-dont-validate]]
- [[total-vs-partial-functions]]

---

```
parse_config("app.toml")
      │
      ├── Ok(config) ──► start(config)
      │
      └── Err(e) ──────► eprintln!("failed: {e}")

throw: скрытый goto       Result: ошибка в сигнатуре
```

Вместо `throw` — тип "успех или ошибка". Ошибка становится частью сигнатуры.

```rust
// Rust — Result<T, E>
fn parse_config(path: &str) -> Result<Config, ConfigError> {
    // Ok(config) или Err(error)
}

match parse_config("app.toml") {
    Ok(config) => start(config),
    Err(e) => eprintln!("failed: {e}"),
}
```

```go
// Go — неформальный Result через multiple return
config, err := parseConfig("app.toml")
if err != nil {
    log.Fatal(err)
}
```

```ts
// TypeScript — можно эмулировать через discriminated union
type Result<T, E> =
    | { ok: true; value: T }
    | { ok: false; error: E }
```

## Где встречается

Rust `Result<T, E>`, Go `(value, err)`, Haskell `Either`, Swift `throws`. В TypeScript/Java — try/catch, но тренд в сторону явных типов ошибок.

## Зачем

`throw` — скрытый goto. Ошибка может прилететь откуда угодно, и компилятор не заставит её обработать. Result делает ошибку видимой в типе — забыть невозможно.

## Происхождение

ML (1973) → Rust (2015), Go `(val, err)`.
