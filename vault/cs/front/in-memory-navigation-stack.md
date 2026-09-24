---
title: In-Memory Navigation Stack
tags: [pattern, navigation, mobile, web]
date: 2026-04-10
---

# In-Memory Navigation Stack

moc: [[front-moc]]
next: [[chrome-mode-fsm]] [[stack-vs-step-navigation]] [[rest-api]]

---

```
  push(C)                pop()
     │                     │
     ▼                     ▼
  ┌─────┐              ┌─────┐
  │  C  │  ← top       │  B  │  ← top
  ├─────┤              ├─────┤
  │  B  │              │  B  │
  ├─────┤              ├─────┤
  │  A  │              │  A  │
  └─────┘              └─────┘
  после push(C)        после pop()   BACK = pop → предыдущий экран
```

**In-memory navigation stack** (он же "view stack") — стандартный паттерн навигации, где экраны/оверлеи хранятся как массив в памяти с операциями push/pop.

## Реализации в фреймворках

| Фреймворк | Как устроено |
|---|---|
| **iOS NavigationStack** | In-memory массив view controllers, push/pop |
| **Android Jetpack Navigation** | NavController с back stack, диалоги — тоже destination на стеке |
| **Flutter Navigator 2.0** | Декларативный `List<Page>` — ближайший аналог `PopupView[]` через reducer |
| **React Navigation (RN)** | `createStackNavigator` — массив route entries в state, push/pop/replace |

## Когда использовать

- **Мобилка** — стандарт де-факто, все фреймворки реализуют нативно
- **Веб (модалы/оверлеи)** — прагматичный выбор: никто не букмаркает модалку, URL-routing здесь избыточен
- **Modal-on-modal** (Info → Terms → back) — единый `PopupView[]` стек, аналогично Android-модели (один back stack для всего)

## Альтернативные подходы

### URL-based routing
Доминирует на вебе. Каждый экран — уникальный URL. Плюсы: букмарки, deep linking, SSR. Минусы: оверлеи/модалы плохо ложатся в URL-модель.

### State machine navigation
Навигация описывается как конечный автомат (XState, statecharts). Переходы между экранами — это transitions между states. Плюсы: формальная верификация, явные допустимые переходы. Минусы: сложность растёт экспоненциально с количеством экранов.

### Coordinator / Router pattern
Отдельный объект (координатор) управляет переходами, экраны не знают друг о друге. Плюсы: decoupling, переиспользование экранов. Минусы: дополнительный слой абстракции.

### Tab-based navigation
Параллельные стеки для каждого таба (iOS UITabBarController, bottom navigation). Каждый таб — независимый navigation stack.

### Graph-based navigation
Экраны — узлы графа, переходы — рёбра. Используется в сложных flow (onboarding, checkout). Пример: Jetpack Navigation с nav graph.

## Derivation-to-coordinator layer

Нестандартная часть — адаптер между стеком и плоской overlay visibility системой (координатором). Нужен когда координатор всё ещё управляет transient UI (tag-filter, location-picker), а стек — основной навигацией.
