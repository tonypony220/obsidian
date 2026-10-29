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

## Дальше (план)

- [[chunking]] — зачем резать документы и что теряется на границах
- [[reranking-cross-encoder]] — второй этап: bi- vs cross-encoder
- [[rag]] — retrieve → generate
- [[retrieval-metrics]] — recall@k, MRR, nDCG, evidence recall
- [[query-decomposition-multi-hop]] — вопрос → подвопросы
- [[structured-extraction]] — схема + цитаты
- [[neuro-symbolic-evaluation]] — LLM читает, код решает
- [[abstention-needs-review]] — «нет доказательства» ≠ «доказано, что нет»
