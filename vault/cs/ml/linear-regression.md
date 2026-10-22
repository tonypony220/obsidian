---
title: Линейная регрессия
tags: [concept, ml, regression]
date: 2026-10-21
---

# Линейная регрессия

moc: [[ml-moc]]
next:
- [[mse]]
- [[least-squares-from-likelihood]]
- [[gradient-descent]]
- [[logistic-regression]]
- [[learning-rate]]

---

![[lr-residuals.png]]

**TL;DR:** подбираем прямую $\hat y = m x + b$ так, чтобы красные отрезки (ошибки) были в сумме как можно меньше; меру «в сумме» даёт [[mse]], подбор — [[gradient-descent]].

## Модель

$$\hat y = m x + b$$

- $m$ (weight) — наклон: насколько растёт $\hat y$ при $x + 1$
- $b$ (bias) — сдвиг: $\hat y$ при $x = 0$
- Много признаков: $\hat y = w_1 x_1 + \dots + w_n x_n + b = w^\top x + b$

## Три шага, которые повторяются везде в ML

| Шаг | Что | Заметка |
|---|---|---|
| 1. модель | $\hat y = m x + b$ | эта |
| 2. ошибка | $J(m,b) = \frac1N \sum (y_i - \hat y_i)^2$ | [[mse]], откуда она → [[least-squares-from-likelihood]] |
| 3. оптимизация | двигать $m, b$ против производной $J$ | [[gradient-descent]] |

## Нормализация признаков

Признаки разного масштаба → производные по весам разного порядка → один [[learning-rate]] не подходит всем.

$$x' = \frac{x - \mu}{\sigma} \qquad \text{или} \qquad x' = \frac{x - \min}{\max - \min}$$

При предсказании нормализовать вход **теми же** $\mu, \sigma$ из train.

## Мост к классификации

[[logistic-regression]] = те же три шага: модель $\sigma(w^\top x + b)$, ошибка log-loss вместо MSE, тот же градиентный спуск.

## Источник

- [ML Cheatsheet — Linear Regression](https://ml-cheatsheet.readthedocs.io/en/latest/linear_regression.html) — по ней учился; обозначения $m$, $b$, пример Radio → Sales
