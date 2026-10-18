---
title: Precision
tags: [concept, ml, metrics]
date: 2026-10-18
---

# Precision

moc: [[ml-moc]]
back: [[confusion-matrix]]
next:
- [[recall]]
- [[f1-score]]
- [[threshold-sweep]]
- [[accuracy-paradox]]

---

```
 модель сказала «positive»:  TP + FP
 из них верно:               TP
 precision = TP / (TP + FP)
```

**TL;DR:** доля верных среди всего, что модель назвала положительным; отвечает на вопрос «насколько можно доверять срабатыванию».

- [[recall]] смотрит с другой стороны: сколько настоящих positive найдено. Precision и recall тянут порог в разные стороны ([[threshold-sweep]]).
- Важен, когда ложная тревога дорога: блокировка аккаунта, автоответ клиенту.
- [[f1-score]] — гармоническое среднее precision и recall.
- Высокий precision при низком recall: модель осторожна, находит мало, но почти не ошибается.
