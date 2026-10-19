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
- [[train-val-test-split]] — три выборки и честная оценка
- [[overfitting]] — выучил шум вместо закономерности
- [[data-leakage]] — информация из будущего в обучении

## Классификация

- [[logistic-regression]] — базовый классификатор, softmax для мультикласса
- [[svm]] — Support Vector Machine, максимизация отступа
- [[random-forest]] — ансамбль деревьев решений
- [[llm-vs-embeddings-classification]] — когда LLM, когда эмбеддинги
- [[text-classification]] — классификация текста: TF-IDF, эмбеддинги, LLM

## NLP и LLM

- [[tokenization]] — текст в токены
- [[transformers]] — attention и генерация
- [[context-window]] — лимит видимого модели
- [[model-parameters]] — веса, размер модели, память
- [[quantization]] — те же веса меньшим числом бит
- [[quantization-number-errors]] — почему страдают числа и даты
- [[moe-mixture-of-experts]] — хранится всё, считается часть
- [[prompting-vs-rag-vs-finetuning]] — три способа дать знания и поведение
- [[information-extraction]] — сущности и структурированные данные

## Эмбеддинги и скоры

- [[embeddings]] — вектор смысла
- [[semantic-search]] — поиск по смыслу
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
- [[precision]] — доля верных среди срабатываний
- [[f1-score]] — гармоническое среднее precision и recall
- [[accuracy-paradox]] — почему accuracy обманывает при дисбалансе

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
