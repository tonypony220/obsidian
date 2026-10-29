---
title: RAG — Retrieval-Augmented Generation
tags: [concept, applied-ai, pattern]
date: 2026-10-29
---

# RAG — Retrieval-Augmented Generation

moc: [[applied-ai-moc]]
back: [[retrieval]]
next:
- [[chunking]]
- [[hybrid-search-rrf]]
- [[reranking-cross-encoder]]
- [[long-context-reading]]

---

```
  ИНДЕКСАЦИЯ (один раз)
  документы ─▶ парсинг ─▶ chunking ─▶ эмбеддинги + BM25-индекс

  ЗАПРОС
  вопрос ─▶ retrieval (hybrid) ─▶ top-50 ─▶ rerank ─▶ top-5
                                                        │
          LLM: "ответь по этим фрагментам" ◀────────────┘
                     │
                   ответ + ссылки на фрагменты
```

**TL;DR:** RAG — LLM отвечает не из своей памяти, а по фрагментам, которые для неё заранее нашёл поиск. R = retrieval (найти), A = augmented (добавить в промпт), G = generation (сгенерировать ответ).

## Зачем

- LLM не знает документов компании: их не было в обучении
- весь корпус не влезает в контекст (или дорого)
- ответ можно привязать к источнику → проверяемо, меньше выдумок

## Стандартный пайплайн

1. [[chunking]] — нарезать документы
2. индексы: [[bm25]] и [[dense-retrieval-embeddings]]
3. [[hybrid-search-rrf]] — кандидаты
4. [[reranking-cross-encoder]] — лучшие
5. LLM отвечает по top-k

Варианты поверх: [[query-decomposition]], [[iterative-retrieval]], [[react-agentic-search]] (agentic RAG).

## Главная слабость

Каждый шаг выбрасывает часть данных: chunking режет, retrieval берёт top-50, reranker — top-5. LLM видит только то, что пережило все фильтры, и **не знает, что что-то пропущено** — уверенно отвечает по неполной картине.

Для вопроса «что сказано про X» это ок. Для «выполнены ли все 4 условия» — опасно: нужен не самый похожий фрагмент, а все нужные. Отсюда альтернативы: [[long-context-reading]], [[exhaustive-scan]].
