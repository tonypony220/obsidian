---
title: "Fail-open vs Fail-closed"
tags: [pattern, tradeoff, error-handling, resilience]
date: 2026-04-12
---

# Fail-open vs Fail-closed

next: [[edge-vs-serverless]]

---

```
Fail-open:    request ──▶ [component FAILS] ──▶ ALLOW  ──▶ downstream
                                                (pass-through)

Fail-closed:  request ──▶ [component FAILS] ──▶ BLOCK  ──▶ error response
```

**Fail-open** — паттерн обработки ошибок, при котором система пропускает (разрешает) запрос когда что-то ломается.

**Fail-closed** — противоположный паттерн: при ошибке система блокирует запрос.

## Примеры

| Сценарий | Fail-open | Fail-closed |
|---|---|---|
| Embedding API недоступен | Сообщение проходит дальше | Сообщение отклоняется |
| Модель не найдена | Фильтр пропускает всё | Фильтр блокирует всё |
| JSON parse error | Считаем "pass" | Считаем "reject" |

## Когда что выбирать

**Fail-open** — когда компонент является оптимизацией, а не критической функцией:
- Лучше обработать "лишнее" сообщение (потратить токены) чем потерять релевантное
- Фильтр падает → сообщения идут на полный LLM → система работает, просто дороже
- Пример: pre-filter для экономии стоимости

**Fail-closed** — когда компонент обеспечивает безопасность или целостность:
- Пример: security filter (spam, abuse), авторизация, валидация платежей
