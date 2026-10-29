---
title: Applied AI MOC
tags: [moc]
date: 2026-10-29
---

# Applied AI

Точка входа: поиск по документам, RAG, извлечение фактов, решения на доказательствах.
Соседний MOC: [[ml-moc]] — основы ML, индексы векторов.

## Поиск (retrieval)

- [[retrieval]] — что такое поиск вообще, семейства методов

- [[bm25]] — лексический поиск: точные слова, редкость слова, длина документа
- [[dense-retrieval-embeddings]] — поиск по смыслу: текст → вектор → косинус
- [[hybrid-search-rrf]] — BM25 + dense, склейка рангов через RRF

## Стратегии сбора доказательств

От «искать мало» к «читать всё»:
- [[query-decomposition]] — сложный вопрос → подвопросы, поиск по каждому
- [[iterative-retrieval]] — несколько раундов, следующий запрос из найденного (multi-hop)
- [[react-agentic-search]] — агент сам выбирает инструменты в цикле
- [[long-context-reading]] — документы целиком в контекст, без поиска фрагментов
- [[exhaustive-scan]] — map-reduce по каждому куску, без top-k

## Решение

- [[neuro-symbolic-evaluation]] — LLM извлекает факты, код решает

## Дальше (план)

- [[chunking]] — зачем резать документы и что теряется на границах
- [[reranking-cross-encoder]] — второй этап: bi- vs cross-encoder
- [[rag]] — retrieve → generate
- [[retrieval-metrics]] — recall@k, MRR, nDCG, evidence recall
- [[structured-extraction]] — схема + цитаты
- [[abstention-needs-review]] — «нет доказательства» ≠ «доказано, что нет»
