---
title: Recall
tags: [concept, ml, metrics]
date: 2026-04-10
---

# Recall

moc: [[ml-moc]]
next: [[fn-rate]] [[f1-score]] [[confusion-matrix]]

---

```
  Все реально Positive:
  ┌──────────────────────────────┐
  │  TP (найдено) ████████████  │  ← Recall = TP / (TP + FN)
  │  FN (пропущено)   ░░░░      │
  └──────────────────────────────┘
  Recall 100% = ничего не потеряно (FN = 0)
```

Из всех реально positive примеров — какую долю модель правильно нашла.

```
Recall = TP / (TP + FN)
```

Recall 100% — ничего не потеряли. Recall 0% — потеряли всё.

Recall — обратная сторона FN rate: `Recall = 1 − FN rate`.

## Precision vs Recall

- **Precision**: из того что модель пропустила — сколько реально positive? (точность)
- **Recall**: из того что реально positive — сколько модель пропустила? (полнота)
