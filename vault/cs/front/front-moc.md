---
title: Frontend MOC
tags: [moc]
date: 2026-04-12
---

# Frontend

Точка входа в заметки по фронтенду.

## Состояние и UI

- [[chrome-mode-fsm]] — FSM для управления chrome-режимами UI
- [[flat-vs-orthogonal-state]] — когда плоское состояние, когда ортогональное
- [[multicast-dispatch]] — N независимых FSM, один dispatch; координация через факт доставки
- [[ui-chrome-vocabulary]] — словарь терминов UI chrome

## Навигация

- [[stack-vs-step-navigation]] — когда стек, когда пошаговый flow
- [[in-memory-navigation-stack]] — навигационный стек в памяти

## Типы и паттерны

- [[interface-vs-discriminated-union]] — два подхода к вариантам типов

## Компоненты

- [[component-layers]] — Primitive / Behavior / Container: разделение по частоте изменений
- [[forward-ref]] — прокидывание ref через компонент к реальному DOM
- [[polymorphic-as-prop]] — один компонент, любой semantic HTML
- [[as-vs-aschild]] — два подхода к полиморфизму: смена тега vs композиция
- [[controlled-vs-uncontrolled]] — кто владеет состоянием компонента
- [[presence-pattern]] — удержание компонента в DOM во время анимации
- [[portal-and-animation-separation]] — Portal/Presence/Sheet как три независимых primitive'а
- [[slot-architecture]] — компонент как набор именованных слотов (compound components)
- [[animate-presence]] — JS-driven Presence из framer-motion (springs, layoutId, gestures)
- [[radix-primitives]] — headless React-компоненты с доступностью из коробки

## Рендеринг и анимация

- [[browser-render-pipeline]] — Style/Layout/Paint/Composite, main vs compositor thread
- [[compositor-layers]] — слои, `will-change`, почему анимации иногда скипаются
- [[animation-approaches]] — уровни анимации от CSS transitions до Canvas, + React Native
- [[request-animation-frame]] — RAF: синхронизация JS-цикла с кадром экрана, vs setInterval
- [[on-exit-complete-callback]] — дождаться exit-анимацию перед следующим действием
- [[pending-action-ref-latch]] — useRef как одноразовая защёлка для отложенного действия
- [[fsm-closing-state]] — exit как first-class состояние FSM, когда FSM владеет данными
- [[fsm-global-transitions]] — cross-kind переходы (cancel/ESC) как catch-all правило в редьюсере

## CSS

- [[css-specificity]] — кортеж `(inline, IDs, classes, elements)`, doubling, отличие от inheritance

## Дизайн-система

- [[design-tokens-sot]] — runtime vs CSS как источник значений токенов

## Архитектура данных

- [[data-layer-sot]] — где живёт state приложения: server / URL / localStorage / Zustand / useState

## Браузерные API

- [[service-worker]] — программируемый прокси между страницей и сетью: offline, push, PWA

## Методология

- [[fix-left-to-right]] — двигайся от дешёвого слоя верификации к дорогому; правки слева удешевляют работу справа
