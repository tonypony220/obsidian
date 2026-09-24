---
title: Cross-compilation
tags: [concept, tool]
date: 2026-07-25
---

# Cross-compilation

moc: [[os-moc]]
next:
- [[freestanding-environment]]

---

```
host: macOS / arm64 ─── i686-elf-gcc ───▶ target: i386 / ELF / без ОС
      (где собираем)                      (где исполняется)

clang (хостовый):  Mach-O, arm64, линкует против macOS SDK   ✗
i686-elf-gcc:      ELF,    i386,  ничего не предполагает      ✓
```

**TL;DR:** host (машина сборки) ≠ target (машина исполнения): ядру нужен ELF/i386 без привязки к какой-либо ОС, а хостовый clang на macOS генерит Mach-O под arm64 — поэтому нужен кросс-тулчейн i686-elf.

## Target triplet

`i686-elf-gcc` читается как `<arch>-<format>`: архитектура i686 (32-битный x86), формат ELF **и никакой ОС** (сравни: `x86_64-apple-darwin`, `x86_64-linux-gnu`). «ELF вместо имени ОС» = bare metal: компилятор не предполагает ни libc, ни syscalls — пара к [[freestanding-environment]]. Полная форма триплета: `<arch>-<vendor>-<os>-<abi>` (например `arm-none-eabi`).

## Что именно отличается

- **Формат бинарника**: Mach-O (macOS) vs [[elf-format|ELF]] — наш [[linker-script]] описывает ELF-секции, GRUB парсит именно ELF.
- **ISA**: arm64 vs i386 — буквально другие машинные инструкции.
- **Предположения об ОС**: хостовый компилятор линкует crt0/libc хоста и генерит код под его ABI и syscalls.

## Где ещё то же самое

- Embedded: `arm-none-eabi-gcc` для микроконтроллеров
- Android NDK / сборка iOS-приложений на mac — кросс-компиляция по определению
- Rust: `--target thumbv7em-none-eabihf` или кастомный target JSON (так будет в Rust-порте KFS)
