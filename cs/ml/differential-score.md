---
title: Дифференциальный скор
tags: [concept, ml, embeddings, classification]
date: 2026-04-10
---

# Дифференциальный скор

moc: [[ml-moc]]
next: [[centroid]] [[threshold-sweep]]

---

```
  msg ──── sim=0.92 ────► centroid_relevant
      \
       \── sim=0.95 ────► centroid_irrelevant

  diff = 0.92 − 0.95 = −0.03  →  irrelevant
```

`diff = sim_relevant − sim_irrelevant`

Разность cosine similarity к двум центроидам. Показывает к какому классу сообщение **относительно** ближе.

- diff > 0 — ближе к relevant
- diff < 0 — ближе к irrelevant

Лучше чем просто `sim_relevant`, потому что учитывает оба центроида. Сообщение с высоким `sim_relevant = 0.92` может иметь ещё более высокий `sim_irrelevant = 0.95` — по одному числу это не видно, по diff (−0.03) — видно сразу.
