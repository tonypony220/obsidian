---
title: MSE — среднеквадратичная ошибка
tags: [concept, ml, regression]
date: 2026-10-21
---

# MSE — среднеквадратичная ошибка

moc: [[ml-moc]]
back: [[linear-regression]]
next:
- [[least-squares-from-likelihood]]
- [[gradient-descent]]

---

![[lr-lines-vs-loss.png]]

**TL;DR:** MSE превращает прямую (пару $m, b$) в одно число «насколько плохо». Квадрат не произвольный: он следует из предположения, что шум в данных гауссов → [[least-squares-from-likelihood]].

## Формула

$$\text{MSE}(m, b) = \frac{1}{N} \sum_{i=1}^{N} \big(y_i - (m x_i + b)\big)^2$$

| Кусок | Смысл |
|---|---|
| $y_i - \hat y_i$ | промах на точке (красный отрезок) |
| $(\cdot)^2$ | убирает знак; большие промахи дороже |
| $\frac1N \sum$ | среднее, не зависит от числа точек |

Важно: данные $x_i, y_i$ фиксированы, меняются только $m, b$. Значит MSE — **функция от параметров**, и у неё есть дно (картинка справа). Это и есть то, от чего берут производную → [[gradient-descent]].

## Откуда $\frac{1}{2}$ (вариант $\frac{1}{2N}$)

$$\frac{d}{de}\, e^2 = 2e \qquad \frac{d}{de}\, \tfrac12 e^2 = e$$

- Двойка из производной сокращается; минимум в той же точке
- Не произвол: $\frac12$ естественно приходит из гауссианы $e^{-\varepsilon^2/2\sigma^2}$ → [[least-squares-from-likelihood]]
- В ML Cheatsheet обе версии: $\frac1N$ в простой, $\frac{1}{2N}$ в многомерной регрессии

## Квадрат vs модуль

- MAE $= \frac1N \sum |e_i|$: излом в 0, производной там нет; следует из шума Лапласа; устойчивее к выбросам
- MSE: гладкая парабола; выбросы доминируют ($e = 10 \to 100$)

## Источник

- [ML Cheatsheet — Cost function](https://ml-cheatsheet.readthedocs.io/en/latest/linear_regression.html#cost-function)
