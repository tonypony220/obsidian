---
title: Parse, Don't Validate
tags: [concept, principle, type-system, pattern]
date: 2026-06-07
---

# Parse, Don't Validate

moc: [[type-design-moc]]
back: [[type-driven-design]]
next:
- [[branded-types]]
- [[make-illegal-states-unrepresentable]]
- [[total-vs-partial-functions]]
- [[result-either]]
- [[zod]]
- [[round-trip-test]]

---

```
validate                                       parse
────────                                       ─────
isEmail(s: string): bool                       parseEmail(s: string): Email | ParseError
        │                                              │
        ▼                                              ▼
проверили, вернули bool                        проверили, вернули доказательство в типе
        │                                              │
дальше по коду снова string                    дальше по коду — Email
        │                                              │
надо проверять снова на каждом стыке           нельзя передать сырой string туда, где ждут Email
"знание потеряно"                              "знание зафиксировано типом"
```

**TL;DR:** `validate :: Raw → bool` теряет знание после проверки — дальше по коду снова `Raw`. `parse :: Raw → Typed | Error` фиксирует знание в типе: получил `Typed` — у тебя в руках **доказательство** валидности, downstream не валидирует заново. Парсить на границе один раз; внутри ядра типы — гарантия.

## Откуда

Alexis King, *«Parse, don't validate»* (2019, Haskell-блог). Прямое применение идеи «использовать систему типов как доказательную систему» к повседневному коду. См. также [[type-driven-design]] для контекста дисциплины.

## Validation vs Parsing

| | Validation | Parsing |
|---|---|---|
| Сигнатура | `Raw → bool` | `Raw → Typed \| Error` |
| Что возвращает | факт «прошло проверку» | объект-доказательство |
| Что с входом дальше | тот же `Raw` тип | сильнее: `Typed` |
| Где живёт знание | в head/комментариях | в типе |
| Сколько раз проверяем | везде, где нужен инвариант | один раз на границе |
| Что говорит компилятор о повторной проверке | ничего | проверка избыточна (тип уже доказательство) |

## Пример

Плохо — validation:
```ts
function isEmail(s: string): boolean { return EMAIL_REGEX.test(s); }

function sendWelcome(email: string) {
  if (!isEmail(email)) throw new Error('bad email'); // снова проверяем
  mailer.send(email);
}

function logSignup(email: string) {
  if (!isEmail(email)) throw new Error('bad email'); // и здесь
  analytics.track('signup', { email });
}
```
- Знание «провалидировано» теряется на каждом стыке
- Сигнатура `string` не отличает «грязный input» от «уже проверенный email»
- Забыть одну проверку — баг, который компилятор не поймает
- Дублирование

Хорошо — parsing:
```ts
type Email = string & { readonly __brand: 'Email' };
type ParseError = { kind: 'ParseError'; reason: string };

function parseEmail(s: string): Email | ParseError {
  return EMAIL_REGEX.test(s)
    ? (s as Email)
    : { kind: 'ParseError', reason: 'invalid email format' };
}

// Граница (route handler) — парсим ОДИН раз
app.post('/signup', (req, res) => {
  const email = parseEmail(req.body.email);
  if ('kind' in email) return res.status(400).json(email);
  sendWelcome(email);
  logSignup(email);
});

function sendWelcome(email: Email) { mailer.send(email); }
function logSignup(email: Email)   { analytics.track('signup', { email }); }
```
- В `sendWelcome` / `logSignup` нельзя передать сырой `string` — компилятор не пустит
- Невалидный email физически не пролетит до бизнес-логики
- Проверка живёт в **одном** месте — на границе
- Сигнатура документирует «здесь требуется уже распарсенный email»

## Границы, на которых парсим

- **HTTP route handler** — request body, query params, path params
- **Webhook consumer** — payload от внешнего сервиса
- **Queue consumer** — message от брокера
- **Form submit** — пользовательский ввод
- **CLI args** — `argv`
- **Config loader** — env vars, файлы конфигурации
- **DB read** — десериализация строки в domain object (хотя БД часто уже даёт схему)

Принцип: **между unstrained world и trusted world** ставится парсер. Дальше «trusted world» работает только с типизированными значениями.

## Что parsing даёт сверх validation

1. **Гарантия неудалимости знания** — компилятор не даст «забыть» инвариант
2. **Документация в типе** — функция, принимающая `Email`, явно требует уже-распарсенное
3. **Сокращение тестов** — не нужны тесты «что если email невалидный в downstream» — этот случай непредставим
4. **Композируемость** — несколько парсеров склеиваются: `parseConfig: Raw → {db: DbConfig, port: Port, ...} | ConfigError`
5. **Single source of truth для формата** — regex/правило живёт **в** парсере, нигде больше

## Антипаттерны

- **Validate then assume** — `if (!isValid(x)) throw; useX(x as ValidX)` — `as` ломает гарантию, тип всё ещё `Raw`
- **Re-validate downstream** — функция принимает `Email`, но внутри снова делает `if (!isEmail(email))` — смелл, говорит что либо не доверяем парсингу, либо лишний код
- **Smart objects without smart constructors** — класс `Email` с публичным конструктором, который бросает: невалидный email **можно** написать, downstream получит exception
- **Двусторонняя валидация без шифровки в тип** — frontend validate + backend validate — обе теряют знание; парсить один раз backend и возвращать типизированный объект

## Связь с другими принципами

- **[[make-illegal-states-unrepresentable]]** — PDV это MILSU применённый к границе системы
- **[[total-vs-partial-functions]]** — после парсинга функции становятся total на доменных типах (`sendWelcome(email: Email)` total, а `sendWelcome(s: string)` partial)
- **[[branded-types]]** — техника для дешёвой реализации PDV в structural-системах (TS)
- **[[result-either]]** — стандартный тип для возврата `Typed | Error`

## Не только в коде

- **Compilers** — лексер/парсер строит AST; downstream-фазы не видят сырой текст. Невозможно «случайно использовать символ, который не лексировался»
- **gRPC/protobuf** — десериализация в типизированный message; handler не видит сырых байт
- **GraphQL** — резолверы получают типизированные аргументы; невалидный input ловится схемой до резолвера
- **JSON Schema / [[zod|Zod]] / io-ts / Pydantic** — runtime-парсинг JSON → typed object. Pydantic для Python — мост к PDV без нативных ADT
- **OS syscalls** — kernel парсит user-space struct в typed kernel object; драйверы не видят raw user memory

Общее везде: **граница системы — место, где сырое становится типизированным**, и обратной дороги (типизированное → сырое в том же flow) нет.
