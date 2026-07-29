---
title: FN rate (False Negative Rate)
tags: [concept, ml, metrics]
date: 2026-04-10
---

# FN rate (False Negative Rate)

moc: [[ml-moc]]
next: [[threshold-sweep]] [[llm-vs-embeddings-classification]]

---

```
  All Positives (TP + FN)
  ┌──────────────────────────┐
  │  TP ✓ ✓ ✓ ✓ ✓ ✓ │ FN ✗ │
  └──────────────────────────┘
                        └──── FN rate = FN / (TP+FN)
```

Процент positive примеров, которые модель ошибочно отнесла к negative.

```
FN rate = FN / (TP + FN)
```

Пример: из 89 relevant сообщений фильтр потерял 1 → FN rate = 1/89 = 1.1%.

Критичная метрика когда потеря positive дороже чем пропуск negative. Например, потерять объявление о жилье хуже, чем пропустить спам в обработку.
