---
title: Классификация текста
tags: [concept, nlp, ml]
date: 2026-10-18
---

# Классификация текста

moc: [[ml-moc]]
back: [[ml-nlp-track]]
next:
- [[llm-vs-embeddings-classification]]
- [[embeddings]]
- [[logistic-regression]]
- [[confusion-matrix]]

---

```
 текст ─► признаки ─► классификатор ─► класс
          (TF-IDF /     (logistic /      (тема, тон,
          эмбеддинг)    SVM / LLM)       намерение)
```

**TL;DR:** текст сводится к вектору признаков и относится к одному из классов, а выбор между TF-IDF, эмбеддингами и LLM задаётся объёмом разметки и ценой ошибки.

| Способ | Нужна разметка | Плюс | Минус |
|---|---|---|---|
| TF-IDF + [[logistic-regression]] / [[svm]] | много | быстро, дёшево, объяснимо | не знает синонимов |
| [[embeddings]] + классификатор / [[centroid]] | средне | понимает смысл | зависит от модели эмбеддингов |
| LLM (zero/few-shot) | почти нет | старт без данных | дороже, медленнее, нестабильнее |

- Детальное сравнение LLM и эмбеддингов: [[llm-vs-embeddings-classification]].
- Оценка: [[confusion-matrix]], [[precision]], [[recall]], [[f1-score]] по каждому классу; дисбаланс классов → [[accuracy-paradox]], [[stratified-split]].
- Дубли текста между train и test — типичная [[data-leakage]].
- Порог уверенности ниже которого «не знаю»: [[threshold-sweep]].
