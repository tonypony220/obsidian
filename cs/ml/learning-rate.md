---
title: Learning rate
tags: [concept, ml, training]
date: 2026-04-12
---

# Learning rate

moc: [[ml-moc]]
next: [[epochs]] [[lambda-regularization]] [[logistic-regression]]

---

```
  loss
   │ ╲
   │  ╲  lr too big: ──→──→── overshoots
   │   ╲         ↗        ↘↗  (diverges)
   │    ╲  lr ok: ──→──→──★  (converges)
   │     ╲ lr tiny:→→→→→→→→→ (too slow)
   └─────────────────────────── w
```

Размер шага при обновлении весов. На каждой итерации модель считает ошибку и сдвигает веса в сторону уменьшения ошибки. Learning rate определяет насколько сильно сдвигать.

- Слишком большой — модель перескакивает оптимум, не сходится
- Слишком маленький — сходится, но очень медленно
- Подобранный — сходится быстро и точно
