---
title: F1 score
tags: [concept, ml, metrics]
date: 2026-04-10
---

# F1 score

moc: [[ml-moc]]
next: [[recall]] [[fn-rate]]

---

```
              2
  F1 = ───────────────────
        1/Precision + 1/Recall

  Precision ──┐
               ├──► F1 (harmonic mean)
  Recall    ──┘
```

Гармоническое среднее precision и recall. Штрафует за перекос — оба числа должны быть высокими.

```
F1 = 2 × (precision × recall) / (precision + recall)
```

Зачем: precision 100% легко — пропусти одно сообщение, но точно relevant. Recall 100% легко — пропусти вообще всё. Оба варианта бесполезны. F1 заставляет балансировать.

- Precision 88.9%, recall 98.9% → F1 = 0.936 (оба хороши)
- Precision 100%, recall 15.7% → F1 = 0.272 (recall провалился)
