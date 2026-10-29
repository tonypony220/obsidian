---
title: Agentic / ReAct search
tags: [concept, applied-ai, retrieval, pattern]
date: 2026-10-29
---

# Agentic / ReAct search

moc: [[applied-ai-moc]]
back: [[iterative-retrieval]]
next:
- [[long-context-reading]]
- [[exhaustive-scan]]

---

```
  ┌─▶ Thought:  нужен отчёт о выбросах за 2024
  │   Action:   list_files("carbon")
  │   Observation: ["Carbon Footprint FY2024.pdf", ...]
  │
  │   Thought:  открою стр. про Scope 3
  │   Action:   read_pages("Carbon...FY2024.pdf", 6-8)
  │   Observation: "категории 1,3,5,6,7 ... 8-15 n/a"
  │
  │   Thought:  нет 2 и 4. Поищу по всему dataroom
  └── Action:   grep("capital goods|category 2")
      ...
      Action:   finish(answer)
```

**TL;DR:** LLM-агент в цикле сам решает, какой инструмент вызвать дальше (поиск, открыть файл, прочитать страницы, grep, посчитать), смотрит на результат и решает снова. ReAct = Reasoning + Acting: мысль → действие → наблюдение → повтор.

## К чему относится

Следующий шаг после [[iterative-retrieval]]: там LLM выбирает только *следующий запрос*, здесь — *любое действие*. Retrieval становится одним из инструментов агента. Статья-источник: ReAct (Yao et al., 2022). Так работают Claude Code, Deep Research-агенты.

## Инструменты для dataroom

| Инструмент | Зачем |
|---|---|
| `list_files` / каталог | понять, что вообще есть |
| `search` (BM25 / dense) | найти фрагменты |
| `read_pages(file, pages)` | прочитать целиком, а не фрагмент |
| `grep` | точная строка, код |
| `read_sheet` | таблица Excel/CSV |
| `python` | посчитать, сконвертировать единицы |

## Плюсы и минусы

| Плюсы | Минусы |
|---|---|
| гибко: подходит к незнакомым целям | недетерминированно: два запуска → разные пути |
| сам переходит к чтению целиком, если фрагментов мало | дорого: много вызовов LLM |
| может перепроверить себя | **решает остановиться сам** → полнота не гарантирована |

Последний пункт главный: агент ищет, пока ему *кажется*, что хватит. Проверить, что он ничего не пропустил, нельзя. Гарантию полноты даёт только [[exhaustive-scan]].
