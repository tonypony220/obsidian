---
title: Cross-validation
tags: [concept, ml, evaluation]
date: 2026-04-12
---

# Cross-validation

moc: [[ml-moc]]
next: [[stratified-split]] [[lambda-regularization]] [[f1-score]]

---

```
  Fold 1: [TEST][    ][    ][    ][    ]
  Fold 2: [    ][TEST][    ][    ][    ]
  Fold 3: [    ][    ][TEST][    ][    ]
  Fold 4: [    ][    ][    ][TEST][    ]
  Fold 5: [    ][    ][    ][    ][TEST]
           └──────── train ────────┘
```

Способ надёжной оценки модели. Данные делятся на K частей (обычно 5). Модель обучается K раз — каждый раз одна часть тестовая, остальные тренировочные. Итог — среднее по K раундам.

```
Раунд 1: [test][train][train][train][train] → F1=0.91
Раунд 2: [train][test][train][train][train] → F1=0.88
...
Среднее: F1 = 0.90 ± 0.02
```

Надёжнее одного случайного сплита, потому что каждый пример побывает в тесте ровно один раз.

**GridSearchCV** — перебирает гиперпараметры (lambda, learning rate) и для каждой комбинации делает cross-validation. Выбирает лучшую автоматически.
