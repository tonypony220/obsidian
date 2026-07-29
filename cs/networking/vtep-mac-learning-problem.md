---
title: VTEP MAC Learning Problem
tags:
  - concept
  - networking
  - overlay
date: 2026-04-16
---

# VTEP MAC Learning Problem

moc: [[networking-moc]]
back: [[vxlan]]
next: [[vxlan-modes]] [[control-plane-data-plane]] [[overlay-underlay]] [[l2-vs-l3]] [[bum-traffic]] [[linux-bridge]]

---

```
   ┌──────────── LOCAL (bridge) ───────────┐   ┌────── UNDERLAY (L3) ──────┐
   │                                       │   │                           │
   │   свои хосты                          │   │   другие VTEP'ы           │
   │   src MAC виден на входе              │   │   видны только как IP     │
   │   обычный L2-learning работает        │   │   "портов" нет            │
   │   → FDB сама заполняется              │   │   → learning не работает  │
   │                                       │   │                           │
   └───────────────────┬───────────────────┘   └─────────────┬─────────────┘
                       │              VTEP                   │
                       └────────────────┬────────────────────┘
                                        ▼
                      вопрос: inner dst MAC X
                              → outer dst IP какого VTEP'а?
                      ответа в VXLAN-протоколе НЕТ
```

**TL;DR:** VXLAN — это только формат инкапсуляции, без механизма discovery. VTEP стоит на стыке локального bridge (где обычный L2-learning работает) и underlay IP-сети (где нет "портов" и learning невозможен). Mapping `inner dst MAC → outer dst VTEP-IP` ниоткуда сам не берётся. Три [[vxlan-modes|режима VXLAN]] — это три способа этот mapping получить.

## Где именно пробел

VTEP выполняет одну техническую роль: **encap/decap**. Он кладёт inner L2-кадр внутрь UDP-пакета с outer-заголовками и отправляет на **какой-то outer dst IP** в underlay. Весь вопрос — откуда взять этот outer dst IP.

Разберём по источникам знания:

### 1. Сам VXLAN-протокол — не помогает

В VXLAN-заголовке ровно одно значащее поле: **VNI** (плюс флаг, что VNI присутствует). Ни discovery, ни announcement, ни sync — в RFC 7348 ничего этого нет. VXLAN — просто конверт.

### 2. Underlay — не помогает

Underlay видит пакеты как обычный IP-трафик между VTEP'ами. Что внутри конверта (какой inner MAC, какой VNI) — underlay не читает и не должен. Это прямое следствие разделения [[overlay-underlay|overlay/underlay]]: underlay специально "тупой", чтобы масштабироваться L3-маршрутизацией.

### 3. Локальный L2-learning — работает только для локальных хостов

На **local-стороне** VTEP — это обычный свитч, подключённый к [[linux-bridge|bridge]] рядом со своими хостами. Классический [[l2-vs-l3|L2]] mechanism работает без изменений: приходит кадр от локального хоста → bridge видит src MAC и порт → FDB пополнилась. Это покрывает «куда слать кадры **к** локальным хостам изнутри».

Но **remote-сторона** VTEP — не свитч: там нет физических портов, есть только IP-туннели до других VTEP'ов. Underlay сам по себе не кидает кадры с src MAC на "порт" VTEP'а — туда приходят только **готовые VXLAN-пакеты**, и то только если их кто-то уже послал именно сюда.

Отсюда **курица-и-яйцо локального learning'а**:

```
  чтобы узнать "MAC X за VTEP_Y"
         │
         ▼
  надо увидеть кадр от MAC X, пришедший из underlay от IP VTEP_Y
         │
         ▼
  но такой кадр придёт только если X сначала пошлёт что-то в ответ
         │
         ▼
  а чтобы X получил исходный кадр, VTEP_Y должен уже знать куда слать
         │
         ▼
  замкнулось
```

Разорвать петлю можно только двумя способами:

- **Забрасывать первый кадр в BUM-флуд** (это делает multicast-режим): если не знаешь куда — отправь **всем**, настоящий получатель ответит, и по ответному кадру запомни путь. Это классический [[bum-traffic|flood-and-learn]], перенесённый в overlay через IP-multicast.
- **Раздать mapping заранее из отдельного источника** — static FDB или BGP EVPN.

### 4. Внешний источник знания — единственный выход

Любой работающий VXLAN-режим так или иначе снабжает VTEP таблицей `MAC → remote VTEP-IP` и списком `VNI → {member VTEPs}`. Три канонических способа ([[vxlan-modes]]):

```
  STATIC     : админ пишет FDB руками
               bridge fdb append <MAC> dev vxlanN dst <remote-VTEP-IP>

  MULTICAST  : IP-multicast + flood-and-learn через underlay
               (VNI → multicast group)  делегирует discovery underlay'ю
               MAC → VTEP  учится из ответного трафика

  EVPN       : BGP распространяет MAC/VTEP-map заранее
               (Type 2 → MAC/IP, Type 3 → VNI membership)
```

## Суть в одной фразе

VTEP учит **свои** MAC'и сам, как обычный L2-свитч. Но он **не может** сам узнать, **за каким удалённым VTEP'ом живёт чужой MAC**, потому что эта информация принципиально находится по ту сторону underlay-туннеля, а underlay о L2 ничего не знает — по [[overlay-underlay|определению]]. Эту дыру закрывает выбранный режим VXLAN.

## Связь с [[control-plane-data-plane]]

Та же проблема в других словах: **у VXLAN нет собственного control plane**. Data plane (encap/decap) описан в RFC 7348, а CP — вынесен наружу. Каждый режим = подстановка разного CP:

- static → CP нет, карта в head человека
- multicast → CP подменён свойством underlay (IP-multicast + learning в data plane)
- EVPN → настоящий CP (BGP) с собственным протоколом распространения состояния
