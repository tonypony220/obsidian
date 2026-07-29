---
title: Controlled vs Uncontrolled — кто владеет состоянием компонента
tags: [concept, pattern, react, frontend]
date: 2026-04-21
---

# Controlled vs Uncontrolled

moc: [[front-moc]]
back: [[component-layers]]
next:
- [[presence-pattern]]
- [[use-imperative-handle]]

---

```
  Controlled                         Uncontrolled

  ┌─ Parent ────────┐                ┌─ Parent ────────┐
  │  [value] ───────┼──▶             │                 │
  │ ◀───[onChange]──┤                │                 │
  └─────────────────┘                └─────────────────┘
          │                                   │
          ▼                                   ▼
  ┌─ Input ─────────┐                ┌─ Input ─────────┐
  │    (зеркало)    │                │    [value]      │
  └─────────────────┘                └─────────────────┘

  state живёт у родителя              state живёт внутри
```

**TL;DR:** Controlled — родитель держит state (`value` + `onChange`); uncontrolled — компонент сам (`defaultValue`). Primitive в дизайн-системе поддерживает оба режима через проверку `value !== undefined`. Переключать режим в runtime нельзя.

## Проблема

Любой интерактивный примитив (input, checkbox, select, combobox) где-то хранит текущее значение. Вопрос — **где**:

1. Внутри компонента — свой `useState`. → **uncontrolled**.
2. У родителя — передаётся пропом, возвращается callback'ом. → **controlled**.

## API двух режимов

```tsx
// Uncontrolled: defaultValue — начальное значение, дальше сам
<input defaultValue="hello" />
const r = useRef<HTMLInputElement>(null);
r.current?.value;  // прочитать — через ref

// Controlled: value + onChange
const [val, setVal] = useState('');
<input value={val} onChange={(e) => setVal(e.target.value)} />
```

## Когда controlled

Нужно **реагировать на каждое изменение**:
- Валидация при вводе (длина, формат)
- Фильтрация списка в combobox/search
- Синхронизация полей («повторите пароль»)
- Счётчик символов, «dirty» state
- Программное изменение — «очистить», автозаполнение

## Когда uncontrolled

Значение нужно **только на submit**:
- Простая форма: заполнил → submit → `FormData` или ref
- Debug-тулбар, быстрый внутренний UI
- Меньше ререндеров (controlled перерисовывает родителя на каждый keystroke)

## Правило для primitive

В дизайн-системе ты не знаешь, что решит пользователь библиотеки. Поддержи оба режима:

```tsx
function Input({ value, defaultValue, onChange }: Props) {
  const [inner, setInner] = useState(defaultValue ?? '');
  const isControlled = value !== undefined;
  const current = isControlled ? value : inner;

  const handle = (next: string) => {
    if (!isControlled) setInner(next);  // сами обновимся
    onChange?.(next);                    // родителю сообщим всегда
  };

  return <input value={current} onChange={(e) => handle(e.target.value)} />;
}
```

Правило: `value !== undefined` → controlled, не трогаем `inner`. Только `defaultValue` → живём сами, `onChange` всё равно эмитим (может быть полезно родителю).

Так устроены `Input` в shadcn/ui, Radix, MUI, Mantine.

## Подводный камень: не переключать режим

```tsx
// ❌ Начали uncontrolled, потом стали controlled
const [val, setVal] = useState<string | undefined>(undefined);
<Input value={val} onChange={setVal} />
// первый рендер: value === undefined → компонент решил "uncontrolled"
// потом val → строка → компонент видит value → переключает режим
// → React warning, state может потеряться
```

Фикс: `value={val ?? ''}` — всегда строка, режим не меняется.

## Tradeoff controlled

**Минус**: ререндер родителя на каждом keystroke. В форме на 20 полей ввод в одно → ререндер всех. При сложной разметке — лаг.

Способы смягчить:
- Держать state рядом с инпутом (отдельный компонент-обёртка), не в корне формы.
- Библиотеки форм (react-hook-form): точечная подписка, без глобального ререндера.
- Валидация на `onBlur`, а не `onChange`.

## Когда обязательно controlled

- **Combobox / Autocomplete** — фильтруешь список по value.
- **Multi-step форма** — значения переносятся между шагами.
- **Live-валидация**.

## Концепция шире

Та же дихотомия «кто источник правды» встречается везде:

- **Data binding**: one-way (React, Flux) vs two-way (Angular, Vue v-model) — кто владеет state: родитель или компонент.
- **Push vs pull**: Observable (push, источник инициирует) vs Iterator (pull, потребитель запрашивает).
- **Hardware bus**: master-slave — один владеет шиной, другой отвечает.
- **Git**: centralized (SVN — сервер SOT) vs distributed (Git — у каждого клона полная история).

Общее везде: **кто держит источник правды и кто является зеркалом** — меняется кто инициирует обновления и где хранится реальное значение.
