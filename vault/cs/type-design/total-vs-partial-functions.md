---
title: Total vs Partial Functions
tags: [concept, principle, type-system]
date: 2026-06-07
---

# Total vs Partial Functions

moc: [[type-design-moc]]
back: [[type-driven-design]]
next:
- [[parse-dont-validate]]
- [[result-either]]
- [[option-maybe]]
- [[branded-types]]

---

```
partial: сигнатура врёт                 total: сигнатура честна
─────────────────────                  ────────────────────────
divide(a, b): number                    divide(a, b: NonZero): number
       │                                       │
   b = 0 → NaN / throw                   b = 0 невозможно по типу
       │                                       │
   "сюрприз в рантайме"                  доказано компилятором

                                        ИЛИ

                                        divide(a, b): Result<number, DivByZero>
                                               │
                                        ошибка — в типе результата
```

**TL;DR:** total функция определена на **всём** своём входном типе и **всегда** возвращает значение из выходного типа. Partial — врёт сигнатурой (на части входов бросает или возвращает мусор). Две стратегии починки: **сузить вход** (branded subtype) или **расширить выход** ([[result-either]] / [[option-maybe]]).

## Определения

- **Total** — для каждого `x: A` значение `f(x): B` существует и не бросает.
- **Partial** — есть `x: A`, на котором `f` бросает, висит или возвращает мусор (NaN, Infinity, undefined behavior).

Partial функция — это **ложь типа**. Сигнатура `(a: number, b: number) => number` обещает «всегда вернёт число»; на `b=0` обещание ломается. Каждый caller должен помнить дополнительный инвариант, которого нет в типе.

## Пример: divide

Partial:
```ts
function divide(a: number, b: number): number {
  return a / b; // b=0 → Infinity или NaN; тип врёт
}
```

### Стратегия А — сузить вход

Используем [[branded-types]] для отсечения недопустимого значения **на границе**:

```ts
type NonZero = number & { readonly __brand: 'NonZero' };

function nonZero(n: number): NonZero | null {
  return n === 0 ? null : (n as NonZero);
}

function divide(a: number, b: NonZero): number { return a / b; }
```

Caller обязан распарсить число через `nonZero()` до вызова. Случай `b=0` отсечён один раз, в месте, где он имеет смысл (например, валидация пользовательского ввода). Дальше по коду — `NonZero` гуляет как доказательство.

### Стратегия Б — расширить выход

Возвращаем ADT, явно кодирующий возможность ошибки:

```ts
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

function divide(a: number, b: number): Result<number, 'DivByZero'> {
  return b === 0
    ? { ok: false, error: 'DivByZero' }
    : { ok: true, value: a / b };
}
```

Caller обязан разобрать `Result` (компилятор не даст обратиться к `.value` без guard'а). См. [[result-either]].

## Когда какая стратегия

| Контекст | Стратегия |
|----------|-----------|
| Невалидный вход — пользовательская ошибка на границе системы | **Сузить вход**: распарсить один раз в `NonZero`/`Email`/`Url`, дальше доказательство гуляет |
| Невалидный вход — динамический случай, неизвестный до момента вызова | **Расширить выход**: `Result<T, E>` рядом с операцией |
| Ошибка — редкое исключение, обработка вверх по стэку через framework | См. [[error-contract-union-vs-throw]] — иногда throw уместнее |

Эвристика: **если знание «вход валидный» можно один раз доказать и переиспользовать** — сужай вход. Если **каждый вызов может провалиться по своей причине** — расширяй выход.

## Total ⊂ Pure

[[pure-functions]] — без побочных эффектов, одинаковый вход → одинаковый выход. Total — определена на всём входе. Это **разные оси**:

- Pure + Total — идеал (`add(1,2) = 3` всегда)
- Pure + Partial — `head([])` в Haskell pure, но partial (бросает)
- Impure + Total — `getCurrentTime(): Date` всегда возвращает что-то, но не pure
- Impure + Partial — `parseJson(s)`-сан-граничной-обёртки — бывает, но самое плохое

TyDD стремится к pure + total в ядре, impure + total на границах. Partial — это «ещё не закончили дизайн типа».

## Red flags

- Сигнатура без `Result`/`Option`/`null`, но функция явно может провалиться (`parseInt`, `JSON.parse`, `Array.at`, `Map.get`)
- `throw` внутри функции с сигнатурой `(...) => T` без union
- Возврат «магических» значений-сентинелов: `-1`, `null`, пустая строка как «не найдено»
- Аннотации `// throws ConfigError` в комментариях
- `!` (non-null assertion) на call-site — caller знает инвариант, которого нет в типе вызываемого

## Не только в функциях

- **API endpoints** — `GET /user/:id` partial (id может не существовать). Total: явный `404` в типе ответа, RFC 7807 Problem Details (см. api-contract-design skill)
- **Database queries** — `SELECT ... WHERE id = X` partial (может вернуть 0 строк). Total: `Option<Row>` в типе результата (Rust sqlx, Haskell persistent)
- **HTTP в браузере** — `fetch().then(r => r.json())` partial (r может быть network error / non-200 / non-JSON). Total: разбор всех ветвей через discriminated union
- **Конструкторы** — `new Email(s)` partial (бросает на невалидном). Total: `Email.parse(s): Email | ParseError` — pattern «smart constructor»

Общее везде: **partial = инвариант в голове**, **total = инвариант в типе**.
