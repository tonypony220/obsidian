---
title: Epochs
tags: [concept, ml, training]
date: 2026-04-12
---

# Epochs

moc: [[ml-moc]]
next: [[learning-rate]] [[lambda-regularization]]

---

```
  Dataset: [d1][d2][d3][d4][d5]

  Epoch 1: → → → → →  loss=0.80
  Epoch 2: → → → → →  loss=0.55
  Epoch 3: → → → → →  loss=0.42
  Epoch N: → → → → →  loss=0.21 ✓
```

Одна epoch = один проход модели по всем тренировочным данным с обновлением весов.

Мало epochs — модель не доучилась (underfitting). Много — модель давно нашла оптимум и крутится на месте. Нужно отслеживать сходимость (падение ошибки по epochs) чтобы понять когда остановиться.
