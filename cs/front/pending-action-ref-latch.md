---
title: useRef как latch для отложенного действия
tags: [pattern, react, state-management, frontend]
date: 2026-04-23
---

# Pending action — ref latch

moc: [[front-moc]]
back: [[on-exit-complete-callback]]
next:
- [[controlled-vs-uncontrolled]]

---

```
момент клика                 момент события
    │                            │
    │  ref.current = action      │
    ├───────────(latch)──────────┤
    │                            │
    │  никаких ре-рендеров       │  onEvent():
    │  между этими точками       │    const a = ref.current
    │                            │    ref.current = null
    │                            │    execute(a)
```

**TL;DR:** когда действие нужно запомнить **сейчас**, но исполнить **позже** — по событию, callback'у или async-резолву — и промежуточный рендер UI не нужен: складывай в `useRef`, не в `useState`. `useRef` не триггерит ре-рендер, это именно механизм, а не отображаемое состояние. Классические кейсы: pending dispatch под `onExitComplete`, одноразовый flag «пользователь кликнул, жди момент», очередь из одного элемента.

## Типовой сценарий

```tsx
const pendingRef = useRef<Profile | null>(null);

const onClick = (p: Profile) => {
  pendingRef.current = p;
  setOpen(false);    // начали exit-анимацию
};

<Sheet
  isOpen={open}
  onExitComplete={() => {
    const p = pendingRef.current;
    pendingRef.current = null;
    if (p) onSelect(p);
  }}
/>
```

Выбранный профиль живёт в ref'е 150 мс между кликом и `onExitComplete`. UI в этот период ничего о нём не должен знать.

## Почему не useState

```tsx
// ❌ плохо
const [pending, setPending] = useState<Profile | null>(null);

const onClick = (p) => {
  setPending(p);        // 1. ре-рендер прямо сейчас
  setOpen(false);       // 2. ещё ре-рендер
};
```

Проблемы:

- **Лишний ре-рендер.** `setPending` тригерит рендер до того, как `isOpen` флипнется. Пустая работа.
- **Соблазн отрендерить промежуточное состояние.** Раз `pending` в state — какой-нибудь view захочет отобразить «Выбрано: X». Это лишняя UI-фаза между клавиатурным ощущением «я кликнул» и моментом прибытия.
- **State превращается в параллельный SOT.** Теперь правда «что пользователь выбрал» живёт одновременно в `pending` и в chrome mode после dispatch'а. Рассинхроны гарантированы.

Ref не вызывает ре-рендер и не торчит в view-дереве — он честно моделирует «это механизм, а не отображаемое состояние».

## Когда всё-таки useState

Если промежуточное UI-состояние **нужно показать**:

```tsx
<span>Выбран: {pending?.name ?? '—'}, ждём закрытия…</span>
```

Тогда `useState` — правильный выбор, потому что данные участвуют в рендере. Вопрос-чеклист: **если я удалю все `{pending}` из JSX, UI-поведение сломается?** Да → `useState`. Нет → `useRef`.

## Что такое latch

Latch (защёлка) — одноразовый контейнер:

- записал значение — защёлка «взведена»
- исполнитель считал и очистил — защёлка «разряжена»
- следующая запись перезатирает предыдущую (новый клик → новое намерение)

Это не очередь — там только один слот. И не буфер — нет batching'а. Чисто «отложить действие до момента X».

Шаблон:

```tsx
// взвести
ref.current = action;

// разрядить (один раз)
const a = ref.current;
ref.current = null;
if (a) execute(a);
```

Очистка сразу после чтения — важна. Иначе при повторном срабатывании callback'а (например, `animationend` из-за side-effect'а) действие исполнится дважды.

## Ref beyond DOM

Типовое заблуждение: `useRef` — только для DOM-узлов. На деле — любой **стабильный mutable контейнер между рендерами**:

- таймеры: `timerRef.current = setTimeout(...)`
- AbortController: `ctrlRef.current = new AbortController()`
- latch'и: `pendingRef.current = action`
- latest-value-ref для stale closure'ов: `latestRef.current = value`
- очереди событий, накопители

Признак что нужен ref: «это должно переживать рендеры, но его изменение **не должно** вызывать ре-рендер».

## Отличие от state-машины

Ref-latch хорош когда:
- caller владеет данными действия
- нужно одно отложенное действие, не дерево переходов
- между `setRef` и исполнением нет промежуточного UI

Когда данных много или exit-анимация **сама должна рендерить эти данные** — ref не годится, нужен [[fsm-closing-state]].

## Концепция шире

«Одноразовый контейнер с side-effect'ом на чтение» — общий паттерн:

- **`std::atomic<bool> once_flag`** в системном программировании — защёлка для single-shot операций.
- **Mailbox / one-shot channel** (Rust `oneshot::channel`, Go `chan T` буфер 1) — одно сообщение, потребитель забирает.
- **Database trigger с `UPDATE … WHERE processed = false` + `SET processed = true`** — взвёл флаг, обработал, разрядил.
- **JWT nonce / CSRF token** — значение одноразовое, после использования недействительно.
- **Redux-thunk pending action** — идея та же, но обёрнута в store.

Общее везде: **атомарная пара «записать намерение сейчас / исполнить ровно один раз потом»**. Latch отделяет момент принятия решения от момента исполнения.
