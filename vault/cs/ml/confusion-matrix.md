---
title: Confusion matrix
tags: [concept, ml, metrics]
date: 2026-04-10
---

# Confusion matrix

moc: [[ml-moc]]
next: [[recall]] [[fn-rate]] [[f1-score]]

---

```
              │ Predicted +  │ Predicted − │
  ────────────┼──────────────┼─────────────┤
  Actual  +   │   TP  ✓      │   FN  ✗     │
  Actual  −   │   FP  ✗      │   TN  ✓     │
```

Таблица 2×2 — четыре исхода классификации:

```
                Predicted +    Predicted -
Actual +            TP             FN
Actual -            FP             TN
```

- **TP** (True Positive) — positive, правильно найден
- **FN** (False Negative) — positive, ошибочно пропущен
- **FP** (False Positive) — negative, ошибочно принят за positive
- **TN** (True Negative) — negative, правильно отсеян

Из этих четырёх чисел считаются все метрики: precision = TP/(TP+FP), recall = TP/(TP+FN), FN rate = FN/(TP+FN).
