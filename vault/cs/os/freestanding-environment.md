---
title: Freestanding environment
tags: [concept]
date: 2026-07-25
---

# Freestanding environment

moc: [[os-moc]]
next:
- [[cross-compilation]]
- [[linker-script]]

---

```
обычная программа        ядро (freestanding)
┌─────────────┐          ┌─────────────┐
│  твой код   │          │  твой код   │
├─────────────┤          ├─────────────┤
│    libc     │          │ (пусто —    │
├─────────────┤          │  всё сам)   │
│     ОС      │          ├─────────────┤
├─────────────┤          │   железо    │
│   железо    │          └─────────────┘
└─────────────┘
```

**TL;DR:** freestanding = код исполняется без libc и ОС под собой; всё, что язык «даёт бесплатно» через runtime, отключается флагами, а нужное (strlen, printf) пишешь сам.

## hosted vs freestanding

Стандарт C различает два окружения. Hosted: полная libc, `main()` особенная. Freestanding: из стандартной библиотеки доступны только заголовки **без кода** — `stdint.h`, `stddef.h`, `stdarg.h`, `stdbool.h` (это заголовки компилятора, не libc — в ядре их использовать можно и нужно). `malloc`, `printf`, `str*` — нет: под ними syscalls, а syscalls делает ОС, которой ты и являешься.

## Что отключает каждый флаг

| Флаг | Что убирает | Почему иначе сломается |
|---|---|---|
| `-ffreestanding` | предположение «есть hosted-окружение» | gcc оптимизирует, считая libc-семантику известной |
| `-fno-builtin` | подмену вызовов встроенными (printf→puts) | твои функции со стандартными именами сматчатся с builtin-семантикой |
| `-nostdlib -nodefaultlibs` | линковку crt0/libc хоста | символы разрешатся в хостовую libc, которая зовёт syscalls несуществующей ОС |
| `-fno-stack-protector` | canary-проверки стека | код canary зовёт `__stack_chk_fail` из runtime, которого нет |
| C++: `-fno-exceptions -fno-rtti` | исключения, dynamic_cast | требуют unwinder и RTTI-runtime |

## Концепция шире

- Embedded/MCU: bare-metal прошивки — то же самое (arm-none-eabi)
- Rust: `#![no_std]` — core вместо std, ровно тот же водораздел
- WASM без WASI: модуль без «ОС», всё внешнее — через импорты

Общее везде: язык делится на «ядро языка» и «runtime, предполагающий ОС»; freestanding оставляет только первое.
