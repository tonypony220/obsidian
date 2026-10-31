---
title: Трек — ML и NLP
tags: [track, ml, nlp, llm]
date: 2026-10-18
---

# Трек — ML и NLP

moc: [[ml-moc]]
next:
- [[ml-task-types]]
- [[tokenization]]

---

```
 Блок A: ML-основы            Блок B: NLP / LLM
 задача → оценка → ловушки    текст → вектор → поиск → модель → способ дать знания

 ml-task-types                tokenization
   → train-val-test-split       → embeddings
   → метрики                      → semantic-search
   → ловушки                      → text-classification / information-extraction
                                  → transformers → context-window
                                  → prompting-vs-rag-vs-finetuning
```

**TL;DR:** упорядоченный маршрут от «какую задачу решает ML и как честно проверить модель» до «как LLM получает знания: prompt, RAG или fine-tuning».

Отметки: `[ ]` не пройдено (ставить `[x]` по мере изучения), 🔗 заметка уже была в vault.

## Блок A. Machine learning algorithms

### A1. Какие задачи решают
- [ ] [[ml-task-types]] — классификация, регрессия, кластеризация
- [ ] 🔗 [[logistic-regression]], [[svm]], [[random-forest]] — примеры классификаторов
- [ ] [[softmax]] — логрег на много классов: логиты → вероятности (мост к LLM)
- [ ] 🔗 [[logreg-to-llm]] — последний слой LLM = та же логрег: связь целиком
- [ ] 🔗 [[centroid]] — простейшая модель «ближайший центр» (пригодится в Блоке B)

### A2. Обучение, валидация, тест
- [ ] [[train-val-test-split]] — три выборки и три вопроса
- [ ] 🔗 [[cross-validation]], [[stratified-split]], [[seed]] — надёжное и воспроизводимое разбиение
- [ ] 🔗 [[learning-rate]], [[epochs]], [[optimizers-gd-vs-lbfgs]], [[derivative]] — как идёт обучение

### A3. Почему хорошая цифра обманывает
- [ ] [[overfitting]] — выучил шум train
- [ ] [[data-leakage]] — ответ просочился в обучение
- [ ] 🔗 [[confusion-matrix]] → [[precision]] → [[recall]] → [[f1-score]] — метрики
- [ ] [[accuracy-paradox]] — дисбаланс классов и ловушки метрик
- [ ] 🔗 [[threshold-sweep]], [[lambda-regularization]] — порог и борьба с переобучением

**Проверка блока A:** объяснить, почему модель с accuracy 99% может быть бесполезна и какие два способа это поймать.

## Блок B. Natural language processing

### B1. Текст → вход модели
- [ ] [[tokenization]] — строка в токены и id

### B2. Смысл как вектор
- [ ] [[embeddings]] — близость векторов = близость смысла
- [ ] [[semantic-search]] — пайплайн поиска по смыслу
- [ ] 🔗 [[hnsw-index]], [[ivf-index]] — приближённый поиск ближайших

### B3. Задачи над текстом
- [ ] [[text-classification]] — TF-IDF vs эмбеддинги vs LLM
- [ ] 🔗 [[llm-vs-embeddings-classification]] — детальное сравнение
- [ ] [[information-extraction]] — сущности и структурированные данные

### B4. LLM
- [ ] [[neural-networks]] — логрег, у которой признаки выучены слоями
- [ ] [[transformers]] — attention и генерация токен за токеном
- [ ] 🔗 [[llm-sampling]] — выход трансформера = softmax по словарю, токен выбирается случайно
- [ ] 🔗 [[temperature]] — деление логитов на T перед softmax
- [ ] [[context-window]] — что модель видит за один вызов
- [ ] [[prompting-vs-rag-vs-finetuning]] — как дать модели знания и поведение

**Проверка блока B:** для задачи «ответы по внутренней базе документов» выбрать prompt / RAG / fine-tuning и нарисовать пайплайн от текста до ответа.

## Порядок и зависимости

| Шаг | Нужно знать заранее |
|---|---|
| [[embeddings]] | [[tokenization]] |
| [[semantic-search]] | [[embeddings]] |
| [[text-classification]] | [[ml-task-types]], [[embeddings]], метрики из A3 |
| [[transformers]] | [[tokenization]], [[embeddings]], [[neural-networks]] |
| [[llm-sampling]], [[temperature]] | [[softmax]], [[transformers]] |
| [[prompting-vs-rag-vs-finetuning]] | [[context-window]], [[semantic-search]] |

## Пробелы

Отдельных заметок пока нет: кластеризация (алгоритмы, k-means), TF-IDF, метрики регрессии. BM25, гибридный поиск, chunking → [[applied-ai-moc]]. Заводить по мере необходимости.
