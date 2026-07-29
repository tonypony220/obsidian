---
title: Engineering Process MOC
tags: [moc]
date: 2026-06-11
---

# Engineering Process — MOC

back: [[architecture-stack]]
next:
- [[enforcement-moc]]
- [[test-strategy-moc]]

---

Слой процесса: как работать, чтобы стек не разъезжался во времени. Самый дорогой слой стека — исполняется людьми, не машинами.

## Порядок работы

- [[left-to-right-fix]] — типы → домен → адаптеры → компоненты; правое не чинит левое
- [[spec-first]] — спека до кода; роадмап = карта спек со статусами, не новая спека
- [[strangler]] — дисциплина наращивается area за area по мере боли, не big-bang rewrite
- [[calibration]] — сколько слоёв оплачивать: cost-of-being-wrong × вероятность × 1/reversibility per layer, не размер проекта

## Инциденты

- [[postmortem-class-sweep]] — инцидент порождает аудит всего класса дефекта, не точечный фикс
- [[audit-by-endpoint]] — producers ищутся grep'ом от endpoint'а по всем поверхностям; grep по папке слеп

## Поставка

- [[pr-boundary-smoke-boundary]] — нарезка PR по стоимости ручной верификации, не по классу работ
