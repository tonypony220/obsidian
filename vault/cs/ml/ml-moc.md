---
title: ML MOC
tags: [moc]
date: 2026-04-12
---

# Machine Learning

Точка входа в заметки по ML.

Трек обучения: [[ml-nlp-track]] — маршрут по ML-основам и NLP/LLM.

## Основы

- [[ml-task-types]] — классификация, регрессия, кластеризация

## Регрессия

- [[linear-regression]] — прямая θᵀx, MSE, градиентный спуск

## Классификация

- [[logistic-regression]] — базовый классификатор, softmax для мультикласса
- [[svm]] — Support Vector Machine, максимизация отступа
- [[random-forest]] — ансамбль деревьев решений
- [[llm-vs-embeddings-classification]] — когда LLM, когда эмбеддинги

## NLP и LLM

- [[optical-context-compression]] — сжатие текста через изображение; исследовательский подход
- [[model-parameters]] — веса, размер модели, память
- [[quantization]] — те же веса меньшим числом бит
- [[quantization-number-errors]] — почему страдают числа и даты
- [[moe-mixture-of-experts]] — хранится всё, считается часть

## Эмбеддинги и скоры

- [[centroid]] — среднее представление класса в пространстве эмбеддингов
- [[hnsw-index]] — многослойный граф для приближённого поиска соседей
- [[ivf-index]] — кластерный индекс с поиском по выбранным спискам
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
