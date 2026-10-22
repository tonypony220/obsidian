---
title: Правило цепочки
tags: [concept, ml, math]
date: 2026-10-22
---

# Правило цепочки

moc: [[ml-moc]]
back: [[derivative]]
next:
- [[partial-derivative]]
- [[gradient-descent]]
- [[linear-regression]]

---

![[derivative-tangent.png]]

**TL;DR:** если функция вложена в функцию $f = A(B(\cdot))$, её производная = **производная внешней × производная внутренней**. Так берётся производная ошибки $(y - (mx + b))^2$ без раскрытия квадрата.

## Вложенные функции

Ошибка на одной точке:

$$f(m, b) = \big(y_i - (m x_i + b)\big)^2$$

Это **две функции, одна внутри другой** (как $\sin(x^2)$: сначала $x^2$, потом $\sin$):

$$\underbrace{B(m, b) = y_i - m x_i - b}_{\text{внутренняя: промах}} \qquad \underbrace{A(B) = B^2}_{\text{внешняя: квадрат}} \qquad f = A\big(B(m, b)\big)$$

## Правило

$$\frac{\partial f}{\partial m} = A'(B) \cdot \frac{\partial B}{\partial m}$$

Интуиция — шестерёнки: если $m$ крутит $B$ с передаточным числом $k_1$, а $B$ крутит $f$ с числом $k_2$, то $m$ крутит $f$ с числом $k_1 \cdot k_2$.

## Применяем (как в ML Cheatsheet)

**1. Внешняя.** $A(B) = B^2 \Rightarrow A'(B) = 2B = 2\big(y_i - (m x_i + b)\big)$

**2. Внутренняя** (по каждой переменной отдельно → [[partial-derivative]]):

$$\frac{\partial B}{\partial m} = -x_i \qquad \frac{\partial B}{\partial b} = -1$$

**3. Перемножаем:**

$$\frac{\partial f}{\partial m} = 2\big(y_i - (m x_i + b)\big) \cdot (-x_i) \qquad \frac{\partial f}{\partial b} = 2\big(y_i - (m x_i + b)\big) \cdot (-1)$$

**4. Среднее по всем точкам** → градиент MSE:

$$\frac{\partial J}{\partial m} = \frac1N \sum_{i=1}^{N} -2 x_i \big(y_i - (m x_i + b)\big) \qquad \frac{\partial J}{\partial b} = \frac1N \sum_{i=1}^{N} -2 \big(y_i - (m x_i + b)\big)$$

## Проверка числами

$x = 2,\ y = 5,\ m = 1,\ b = 0$ → $B = 3$, $f = 9$.

- Формула: $2 \cdot 3 \cdot (-2) = -12$
- Вручную: $m = 1.001$ → $B = 2.998$, $f = 8.988$ → $\frac{-0.012}{0.001} = -12$ ✅

## Концепция шире

- Нейросети: backpropagation = цепочка через все слои, от ошибки назад к каждому весу
- Физика: $\frac{dT}{dt} = \frac{dT}{dh} \cdot \frac{dh}{dt}$ — температура при подъёме
- PyTorch/JAX autograd: граф вычислений, локальные производные перемножаются

Общее везде: сложную зависимость режем на вложенные простые и перемножаем их производные.

## Источник

- [ML Cheatsheet — Gradient descent](https://ml-cheatsheet.readthedocs.io/en/latest/linear_regression.html#gradient-descent)
