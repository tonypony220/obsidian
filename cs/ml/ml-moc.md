---
title: ML MOC
tags: [moc]
date: 2026-04-12
---

# Machine Learning

Точка входа в заметки по ML.

## Классификация

- [[logistic-regression]] — базовый классификатор, softmax для мультикласса
- [[svm]] — Support Vector Machine, максимизация отступа
- [[random-forest]] — ансамбль деревьев решений
- [[llm-vs-embeddings-classification]] — когда LLM, когда эмбеддинги

## Эмбеддинги и скоры

- [[centroid]] — среднее представление класса в пространстве эмбеддингов
- [[differential-score]] — разность расстояний до центроидов
- [[score-distribution]] — распределение скоров модели по классам
- [[threshold-sweep]] — подбор порога по метрикам

## Метрики

- [[confusion-matrix]] — таблица TP/FP/TN/FN
- [[recall]] — доля найденных положительных
- [[fn-rate]] — доля пропущенных положительных
- [[f1-score]] — гармоническое среднее precision и recall

## Обучение

- [[learning-rate]] — шаг градиентного спуска
- [[epochs]] — сколько раз пройти по данным
- [[lambda-regularization]] — штраф за сложность модели
- [[optimizers-gd-vs-lbfgs]] — Gradient Descent vs LBFGS
- [[derivative]] — производная как основа оптимизации

## Оценка

- [[cross-validation]] — k-fold проверка на разных подвыборках
- [[stratified-split]] — сохранение пропорций классов при разбиении
- [[seed]] — фиксация случайности для воспроизводимости
