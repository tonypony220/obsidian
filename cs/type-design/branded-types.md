---
title: Branded Types
tags: [concept, technique, type-system]
date: 2026-06-07
---

# Branded Types (Nominal в Structural-системе)

moc: [[type-design-moc]]
back: [[type-driven-design]]
next:
- [[parse-dont-validate]]
- [[total-vs-partial-functions]]
- [[algebraic-data-types]]

---

```
TS (compile-time fiction)                   Rust (runtime nominal)
─────────────────────────                   ──────────────────────
type Email = string & { __brand: 'Email' }  pub struct Email(String);
                                            //       ^ приватное поле
       │                                            │
       ▼                                            ▼
после tsc → просто string                    реальная struct в памяти
__brand нигде не существует                  поле приватно → снаружи не сконструировать
обход: "garbage" as Email                    обход: невозможен компилятором
защита = дисциплина                          защита = система типов
```

**TL;DR:** structural-системы (TS, Go) считают одинаковые по форме типы взаимозаменяемыми — `UserId` и `OrderId` оба `string`. Branded type добавляет **различающий тег**, чтобы они стали nominally разными. В Rust это реально nominal (struct в памяти). В TS — фикция, существующая только в системе типов: после компиляции тега нет. Защита держится на smart constructor + (опционально) приватном symbol.

## Structural vs Nominal

- **Structural** (TS, Go, OCaml-records): два типа совместимы, если их **форма** совпадает. Имя — комментарий.
- **Nominal** (Rust, Java, C#, Haskell `newtype`): два типа совместимы, только если они **именованно одинаковы**. Форма не достаточна.

TS structural по дизайну: `UserId` и `OrderId` оба `string` — одна и та же «дырка». Brand — искусственное различие, поднимающее nominal-разделение поверх structural-системы.

## TypeScript: type erasure и фантомное поле

### Что такое type erasure

TS-типы существуют **только при компиляции**. После `tsc` они исчезают, остаётся обычный JS:

```ts
type Email = string & { readonly __brand: 'Email' };
const e: Email = "a@b.c" as Email;

console.log(e);          // "a@b.c"
console.log(typeof e);   // "string"
console.log(e.__brand);  // undefined — никакого __brand нет
```

`__brand` нигде не записывается в значение. Это **выдумка для компилятора**, нужная только чтобы он считал `Email` и `string` разными типами.

### Как тип работает

`string & { __brand: 'Email' }` — это **intersection** («и то и то»). TS читает: «Email — это string, у которого ЕЩЁ есть поле `__brand: 'Email'`». Технически это ложь — у строки не может быть `__brand`. Но TS не проверяет «может ли», он сравнивает типы при присваивании:

```ts
const raw: string = "hello";
const e: Email = raw;  // ❌ у string нет __brand → нельзя присвоить Email
```

Это и нужно — компилятор различает «грязный string» и «уже распарсенный Email».

### Как получить Email

Только через `as` (type assertion). Это команда компилятору «поверь мне»:

```ts
const e = "hello" as Email;  // ✅ компилится — наврали компилятору
```

`as` — runtime no-op. В JS он исчезает. Это compile-time-only «костыль доверия».

### Проблема: `as` доступен везде

Если каждый может писать `"garbage" as Email` — гарантия рушится. Тип объявлен, но защиты нет.

Решение — **smart constructor**: договорённость, что `as Email` живёт **только в одной функции**, которая делает проверку.

```ts
// email.ts
export type Email = string & { readonly __brand: 'Email' };

export function parseEmail(s: string): Email | ParseError {
  if (!EMAIL_REGEX.test(s)) return { kind: 'ParseError', reason: 'invalid' };
  return s as Email;  // ← ЕДИНСТВЕННЫЙ cast во всей кодовой базе
}
```

Использование:
```ts
import { Email, parseEmail } from './email';

const e = parseEmail(input);              // ✅ только так получаем Email
if ('kind' in e) return error(e);
sendWelcome(e);                            // e: Email, провалидирован
```

**Что enforces «единственный cast»:**
1. Дисциплина команды — все знают: `as Email` только внутри `parseEmail`
2. `grep "as Email"` находит все cast'ы — должен быть один, на ревью видно
3. ESLint правило может запретить cast в branded-типы вне их модуля

Это **слабая** защита. Технически TS не мешает написать `as Email` где угодно. Защита держится на дисциплине.

### Сильная защита: приватный brand через `unique symbol`

Если нужна **техническая** гарантия, а не договорённость:

```ts
// email.ts
declare const emailBrand: unique symbol;          // приватный symbol
export type Email = string & { readonly [emailBrand]: true };

export function parseEmail(s: string): Email | ParseError {
  if (!EMAIL_REGEX.test(s)) return { kind: 'ParseError' };
  return s as Email;
}
```

`emailBrand` объявлен `declare const ... : unique symbol` и **не экспортируется**. Снаружи модуля он невидим. Поэтому:

```ts
// другой файл
import { Email } from './email';

const e: Email = "garbage" as Email;
// ❌ TS Error: Type 'string' is not assignable to 'Email'.
// Property '[emailBrand]' is missing — но emailBrand вообще не виден снаружи
```

Cast `as Email` снаружи модуля **не сработает** — TS не может ссылаться на приватный symbol. Получить `Email` можно только через `parseEmail`. Это уже настоящая инкапсуляция compile-time.

В рантайме — всё то же, просто строка. Symbol существует только в системе типов как уникальный идентификатор-«ключ».

## Rust: настоящий nominal через newtype

В Rust nominal встроен. Не нужны фантомные поля — `String` и `Email(String)` это **разные типы по имени**, даже с одинаковой формой.

```rust
// email.rs
pub struct Email(String);   // tuple struct; поле без pub → приватное

#[derive(Debug)]
pub enum ParseError { InvalidFormat }

impl Email {
    pub fn parse(s: String) -> Result<Email, ParseError> {
        if !is_valid_email(&s) {
            return Err(ParseError::InvalidFormat);
        }
        Ok(Email(s))  // конструктор Email(...) доступен только тут — поле приватное
    }

    pub fn as_str(&self) -> &str { &self.0 }
}
```

Использование:
```rust
use crate::email::Email;

fn send_welcome(email: Email) { /* ... */ }

let email = Email::parse(String::from("a@b.c"))?;
send_welcome(email);

send_welcome(String::from("hello"));               // ❌ expected Email, found String
let fake = Email(String::from("garbage"));         // ❌ tuple struct constructor is private
```

**Что enforces:**
1. **Реальный тип** — `Email` это struct в рантайме. Компилятор физически не пустит `String` туда, где ждут `Email`.
2. **Приватное поле** (`String` без `pub`) — снаружи модуля **нельзя** написать `Email("trash".to_string())`. Конструктор недоступен.
3. **Единственный путь** — публичный `Email::parse`. Хочешь Email — иди через парсер.

Дисциплина не нужна — компилятор закрывает обходные пути.

### Zero-cost обёртка

Обычный `struct Email(String)` в памяти занимает столько же, сколько `String` (т.к. поле одно). Но для гарантии identical representation — `#[repr(transparent)]`:

```rust
#[repr(transparent)]
pub struct Email(String);
```

Контракт с компилятором: представление в памяти **идентично** внутреннему типу. Используется для FFI и оптимизаций. Email и String байт-в-байт одинаковы в памяти, при этом типы разные.

## Сводная таблица

|  | TS branded | Rust newtype |
|---|---|---|
| Что в рантайме | просто string, никаких следов `__brand` | struct (или transparent — байт-в-байт как String) |
| Различение типов | фикция в системе типов | настоящий nominal-тип |
| Можно ли обойти cast'ом | `as Email` где угодно (public brand); нельзя (private symbol-brand) | нет — приватное поле + закрытый конструктор |
| Стоимость в памяти | 0 (это string) | 0 с `#[repr(transparent)]`, иначе пустяковая обёртка |
| Enforce smart constructor | дисциплина + grep + опц. private symbol | компилятор + модульная инкапсуляция |
| Конвертация | `as Email`, `email as string` — оба бесплатно | `Email::parse(s)?`, `email.0` — явно |

## Реализация в других языках

| Язык | Механизм |
|------|----------|
| **Haskell** | `newtype UserId = UserId String` — буквально то же, ZERO cost |
| **Kotlin** | `value class UserId(val v: String)` (inline classes) — без overhead |
| **Swift** | `struct UserId { let value: String }` — обёртка |
| **Java** | `record UserId(String value)` — runtime-обёртка |
| **Go** | `type UserId string` — даёт nominal, но `string(uid)` свободно конвертит — защита слабая |
| **Python + mypy** | `UserId = NewType('UserId', str)` — mypy-only различие, runtime игнорирует |

В языках с настоящим nominal typing (Rust/Haskell/Kotlin) branded types — нативная и бесплатная конструкция. В structural-языках (TS) — эмуляция через фантомное поле, защита держится на дисциплине или приватных symbol'ах.

## Что хорошо ложится на брендинг

- **Идентификаторы**: `UserId`, `OrderId`, `ProductSku`
- **Форматные строки**: `Email`, `Url`, `Uuid`, `IsoDate`, `IsoCountryCode`, `PhoneE164`
- **Численные подмножества**: `NonZero`, `Positive`, `Percentage` (0–100), `Probability` (0–1)
- **Sanitized строки**: `SafeHtml`, `EscapedSql`, `NormalizedEmail`
- **Currency-amount**: `Cents`, `Usd`, `Eur` — нельзя случайно сложить USD с EUR без явной конвертации
- **Validated tokens**: `JwtToken`, `CsrfToken` — отличается от сырой строки

## Антипаттерны

- **Brand без smart constructor** — `export function asEmail(s: string): Email { return s as Email; }` без проверки. Тип есть, гарантии нет — хуже чем ничего, создаёт иллюзию безопасности
- **Brand для несвязанных типов** — `Brand<string, 'X'>` ради «чтобы было». Если нет реальной class-of-bugs, которую он ловит — шум
- **Утечка brand'а наружу** — экспортировать `__brand` напрямую или давать публичные cast-helpers. Уничтожает инкапсуляцию
- **Rust: `pub` на поле tuple struct** — `pub struct Email(pub String)` — конструктор открывается снаружи модуля, гарантия теряется

## Не только в типах

- **HTTP**: `Bearer <token>` vs `Basic <token>` — формат отличает классы токенов на уровне протокола
- **Email**: `Reply-To` vs `From` vs `Sender` — все строки одного формата, но семантически разные роли; SMTP различает по имени поля
- **Money**: ISO 4217 (`USD`, `EUR`) — тэг валюты неотделим от amount; нельзя складывать без явной conversion
- **Quantities в физике**: метры vs секунды vs кг — оба `float`, операции между ними бессмысленны без unit conversion (F#/Boost.Units enforce этим)
- **DB foreign keys vs random IDs** — FK constraint = nominal-проверка на уровне БД: «эта колонка ссылается на ту таблицу, не любой UUID»

Общее везде: **одинаковая форма ≠ одинаковая семантика**; nominal-различие защищает от подмены классов значений.
