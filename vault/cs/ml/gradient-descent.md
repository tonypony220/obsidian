---
title: Градиентный спуск для линейной регрессии
tags: [concept, ml, training]
date: 2026-10-22
---

# Градиентный спуск для линейной регрессии

moc: [[ml-moc]]
back: [[linear-regression]]
next:
- [[derivative]]
- [[chain-rule]]
- [[mse]]
- [[learning-rate]]
- [[epochs]]
- [[optimizers-gd-vs-lbfgs]]

---

![[gd-steps.png]]

**TL;DR:** ошибка $J$ — функция от параметров $m, b$ (кривая на картинке). Производная $\frac{\partial J}{\partial m}$ = наклон этой кривой: её знак говорит, в какую сторону менять $m$, чтобы ошибка падала. Шаг: $m \leftarrow m - \alpha \frac{\partial J}{\partial m}$.

## Где здесь производная

![[lr-lines-vs-loss.png]]

- Данные $x_i, y_i$ фиксированы. Крутим только $m$ и $b$
- Каждое $m$ → своя прямая → своё значение $J$. Получаем **функцию** $J(m)$ — параболу справа
- Нужно найти дно. Перебирать все $m$ дорого → смотрим на **наклон в текущей точке** ([[derivative]])
- Наклон $> 0$ → справа выше → идём влево (уменьшаем $m$). Наклон $< 0$ → наоборот. Поэтому минус в формуле шага

## Производная — это тоже функция

$J(m)$ — кривая. Её производная $\frac{dJ}{dm}$ — **тоже функция от $m$**: подставил любое $m$ → получил наклон кривой в этой точке. Нам вся она не нужна: каждый шаг считаем наклон **только в текущем $m$**, шагаем, снова считаем в новом.

## Вычисление производной

$m$ влияет на ошибку не напрямую, а цепочкой: $m \to \hat y \to$ промах $\to$ квадрат. Производная по цепочке = произведение множителей звеньев → [[chain-rule]].

Для одной точки $(x_i, y_i)$:

$$\frac{\partial}{\partial m}(y_i - \hat y_i)^2 = \underbrace{2(y_i - \hat y_i)}_{\text{квадрат}} \cdot \underbrace{(-1)}_{\text{промах}} \cdot \underbrace{x_i}_{\hat y = m x + b}$$

$$\frac{\partial}{\partial b}(y_i - \hat y_i)^2 = 2(y_i - \hat y_i) \cdot (-1) \cdot \underbrace{1}_{\hat y = m x + b}$$

Разница только в последнем множителе: насколько $\hat y$ меняется при сдвиге параметра.
- $m$ на $+1$ → $\hat y$ на $+x_i$ (точка с большим $x$ сильнее реагирует на наклон)
- $b$ на $+1$ → $\hat y$ на $+1$ всегда

Усредняем по всем точкам (как в ML Cheatsheet):

$$\frac{\partial J}{\partial m} = \frac1N \sum_{i=1}^{N} -2 x_i \big(y_i - (m x_i + b)\big) \qquad \frac{\partial J}{\partial b} = \frac1N \sum_{i=1}^{N} -2 \big(y_i - (m x_i + b)\big)$$

## Шаг обновления

$$m \leftarrow m - \alpha \frac{\partial J}{\partial m} \qquad b \leftarrow b - \alpha \frac{\partial J}{\partial b}$$

Оба градиента считать по **старым** $m, b$, потом обновлять оба.

## Проверка интуиции на числах

Одна точка $x = 2,\ y = 5$; старт $m = 0,\ b = 0$; $\alpha = 0.1$.

| | значение |
|---|---|
| $\hat y$ | $0$ → прямая ниже точки, промах $y - \hat y = 5$ |
| $\partial J/\partial m$ | $-2 \cdot 2 \cdot 5 = -20$ (отрицательный → $m$ надо увеличить) |
| $\partial J/\partial b$ | $-2 \cdot 5 = -10$ |
| новые | $m = 0 + 2 = 2,\ b = 0 + 1 = 1$ → $\hat y = 5$ |

## Псевдокод

```python
for epoch in range(n_epochs):
    y_hat = m * x + b
    err = y - y_hat
    dm = -2 * mean(x * err)
    db = -2 * mean(err)
    m -= lr * dm
    b -= lr * db
```

## Источник

- [ML Cheatsheet — Gradient descent](https://ml-cheatsheet.readthedocs.io/en/latest/linear_regression.html#gradient-descent)
- [ML Cheatsheet — Gradient Descent (общая)](https://ml-cheatsheet.readthedocs.io/en/latest/gradient_descent.html)
