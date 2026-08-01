---
title: Binary inspection cheatsheet
tags: [tool, cheatsheet]
date: 2026-07-26
---

# Binary inspection cheatsheet

moc: [[os-moc]]
next:
- [[elf-format]]
- [[code-and-data-are-bytes]]

---

```
boot.asm ─nasm─▶ boot.o ──┐
kmain.c ──gcc─▶ kmain.o ──┼──ld──▶ kernel.bin ──qemu──▶ живая VM
                          │
глаза на .o:              │    глаза на kernel.bin:     глаза на VM:
nm, objdump -r            │    nm -n, objdump -h/-s/-d  QEMU monitor
```

**TL;DR:** nm/objdump/readelf — глаза для бинарников (build-time): символы, секции, сырые байты, дизассемблер, заплатки. QEMU monitor — те же глаза, но на работающую машину (run-time): регистры, память. Общий язык двух миров — адреса.

## Сборка руками (до Makefile)

```sh
nasm -f elf32 boot/boot.asm -o boot.o
i686-elf-gcc -ffreestanding -fno-builtin -fno-stack-protector \
    -nostdlib -nodefaultlibs -Wall -Wextra -c src/kmain.c -o kmain.o
i686-elf-ld -T linker.ld -o kernel.bin boot.o kmain.o
```

На школьном Linux: `gcc -m32`, `ld -m elf_i386`, без префиксов (спека §2).

## Инструменты

| Команда | Что показывает |
|---|---|
| `nm -n f` | символы по возрастанию адресов. `U` = дырка, `T/t` = код, `b` = .bss, `a` = константа-equ |
| `objdump -h f` | секции: размер, VMA (адрес в памяти), offset в файле |
| `objdump -s -j .text f` | сырые байты секции. Магию искать перевёрнутой — little-endian! |
| `objdump -d -M intel f` | дизассемблер: байты → мнемоники. Без `-M intel` печатает AT&T-синтаксис: операнды НАОБОРОТ (`mov src, dst`), `%eax`, `$42`; с флагом — Intel, как NASM (`mov dst, src`). Шорткат: `make dump` |
| `objdump -r f.o` | relocations — таблица заплаток «куда вписать финальные адреса» (есть только в .o) |
| `readelf -h f` | ELF-заголовок ([[elf-format]]): entry point, машина, тип |

На маке все с префиксом `i686-elf-` (хостовые ждут Mach-O).

## QEMU monitor — глаза на живую машину

Монитор — служебная консоль QEMU (не экран VM!): вопросы к работающей машине.
В окне QEMU: Ctrl-Alt-2 (обратно Ctrl-Alt-1). Headless-сценарий без окна:

```sh
# subshell печатает команды монитору по расписанию, пайп подаёт их на stdin
(sleep 2; echo 'info registers'; sleep 1; echo 'quit') | \
  qemu-system-i386 -kernel kernel.bin -display none -monitor stdio
```

| Команда монитора | Что показывает |
|---|---|
| `info registers` | мгновенный снимок регистров: EIP = адрес исполняемой инструкции |
| `x /8x 0xB8000` | дамп памяти по адресу (examine); `xp` — по физическому |
| `info mtree -f` | карта «адрес → устройство»: маршрутизация чипсета глазами ([[memory-mapped-io]]) |
| `quit` | выключить VM (наше ядро само не завершится никогда) |

В интерактивном мониторе команды набираются напрямую (`info registers` + Enter,
Tab дополняет, стрелка вверх — история). `echo` из headless-сценария — команда
shell'а, печатавшая текст на stdin QEMU, а не команда монитора.

Метод «где сейчас CPU»: EIP из монитора сопоставить с `objdump -d` / `nm -n` —
динамика (что машина делает СЕЙЧАС) встречается со статикой (что мы записали
в файл при сборке) на общем языке адресов. Так доказано «ядро живо» в этапе 1:
EIP=0x100021 = адрес `nop` внутри цикла kmain.

## Что такое .o на самом деле

Три части: **байты секций** + **таблица символов** (имена → позиции; их пишет ассемблер — так линкер «знает слово stack_top») + **таблица заплаток** (relocations: «в байте N дырка под адрес символа X»). После линковки имена инструкциям не нужны — в них вшиты числа; symtab остаётся для отладки (`strip` уберёт — ядро продолжит грузиться).
