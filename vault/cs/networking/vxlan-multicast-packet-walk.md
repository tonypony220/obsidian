---
title: VXLAN Multicast — Packet Walk
tags:
  - concept
  - networking
  - overlay
  - cheatsheet
date: 2026-04-16
---

# VXLAN Multicast — Packet Walk

moc: [[networking-moc]]
back: [[vxlan-modes]]
next:
- [[ip-multicast]]
- [[igmp]]
- [[pim]]
- [[vtep-mac-learning-problem]]
- [[bum-traffic]]
- [[vxlan]]
- [[arp]]
- [[vxlan-bridge-setup]]

---

```
   host_A                                                            host_B
   30.1.1.1  ─── eth1 ┐                                      ┌─ eth1 ──── 30.1.1.2
                       │                                      │
                    [VTEP-A]                               [VTEP-B]
                  routeur_wil-1                          routeur_wil-2
                 vxlan10 VNI=10                         vxlan10 VNI=10
                 group 239.1.1.1                        group 239.1.1.1
                       │                                      │
                 eth0 10.0.0.1 ────┐                ┌──── eth0 10.0.0.2
                                   │                │
                           ┌───── R_core (RP) ─────┐
                           │      10.0.0.250       │
                           │   запущен PIM-SM      │
                           │   static RP = self    │
                           └───────────────────────┘

   underlay — 10.0.0.0/24, L3, с PIM-SM; unicast-роутинг — OSPF/static
```

**TL;DR:** Практическая трасса ARP-broadcast'а от `host_A` к `host_B` через VXLAN в multicast-режиме. Показывает **Phase 0** — что должно быть заранее построено в underlay ([[igmp|IGMP]] Report → [[pim|PIM]] Join → RPT), **Phase 1–5** — путь первого BUM-пакета (encap → PIM Register / RPT / RPF → decap → flood-and-learn на удалённом VTEP'е), **Phase 6** — как ответ идёт уже обычным unicast'ом без участия multicast. Главная мораль: multicast в underlay'е работает только на «знакомство» VTEP'ов, дальше трафик переходит в unicast.

## Топология для трассы

Минимальная осмысленная установка, параллельная BADASS P2, плюс явно выделенный RP для полноты. См. схему выше. Цель трассы: `host_A` пингует `host_B`. Первым делом идёт **ARP Request** на `30.1.1.2` — это и есть наш BUM-кадр.

---

## Phase 0 — что УЖЕ произошло до нашего пакета

Это половина всего понимания. Чтобы первый BUM-пакет доехал, в underlay должна быть заранее отработана цепочка из двух протоколов.

```
  1. ip link add vxlan10 type vxlan id 10 group 239.1.1.1 dev eth0
     ───────────────────────────────────────────────────────────────
     → kernel VXLAN-модуль делает IP_ADD_MEMBERSHIP на 239.1.1.1
     → kernel шлёт IGMP Membership Report → в underlay, на eth0

  2. R_core (он же first-hop router для обоих VTEP'ов) получает IGMP Report
     от VTEP-A и VTEP-B:
     → создаёт запись multicast FIB: (*, 239.1.1.1)
         OIL = [интерфейс к VTEP-A, интерфейс к VTEP-B]

  3. R_core = RP, поэтому PIM Join никуда дальше не идёт
     (если бы RP был на другом роутере — R_core послал бы Join upstream
      к RP, и запись (*, G) появилась бы по всей цепочке hop-by-hop)
```

**Состояние после Phase 0:** shared-tree (RPT) для `239.1.1.1` построен. RP знает: «придёт что-то на эту группу — лить в обе стороны».

В этот момент трафика ещё ноль — это чисто control-plane активность, отработавшая при boot'е.

---

## Phase 1 — host_A шлёт ARP Request

```
   host_A (30.1.1.1)
       │
       │ ARP Request: кто такой 30.1.1.2?
       │
       │ Ethernet кадр:
       │   src MAC = aa:bb:cc:00:00:01  (host_A)
       │   dst MAC = ff:ff:ff:ff:ff:ff  (broadcast)
       │   payload = ARP "who has 30.1.1.2 tell 30.1.1.1"
       │
       ▼
   eth1 VTEP-A (в bridge br0)
```

Обычный broadcast кадр. Попадает в `br0` на VTEP-A.

---

## Phase 2 — VTEP-A инкапсулирует в VXLAN

`br0` смотрит на `dst MAC = ff:ff:ff:ff:ff:ff` → это [[bum-traffic|BUM]], флуд на все порты кроме входного. В `br0` есть два порта: `eth1` (откуда кадр и пришёл) и `vxlan10`. Флудится на `vxlan10`.

`vxlan10` как VTEP-интерфейс получает inner-кадр и решает куда слать:

```
  vxlan10 в MULTICAST-режиме:
  "для BUM — слать на group 239.1.1.1"  (задано через group при ip link add)

  Собирает outer-пакет:
  ┌───────────────────────────────────────────────┐
  │ OUTER Eth                                     │
  │   src MAC = MAC VTEP-A eth0                   │
  │   dst MAC = 01:00:5e:01:01:01  (mcast MAC    │  ← mapping 239.1.1.1
  │                                 для 239.1.1.1)│    в multicast MAC
  ├───────────────────────────────────────────────┤
  │ OUTER IP                                      │
  │   src IP = 10.0.0.1      (VTEP-A)             │
  │   dst IP = 239.1.1.1     (multicast group)    │
  ├───────────────────────────────────────────────┤
  │ OUTER UDP  dst port = 4789                    │
  ├───────────────────────────────────────────────┤
  │ VXLAN header  VNI = 10                        │
  ├───────────────────────────────────────────────┤
  │ INNER Eth (оригинальный кадр, нетронутый)     │
  │   src MAC = aa:bb:cc:00:00:01 (host_A)        │
  │   dst MAC = ff:ff:ff:ff:ff:ff (broadcast)     │
  │   payload = ARP Request                       │
  └───────────────────────────────────────────────┘

  → отправляется в underlay через eth0
```

Ключевое: с точки зрения underlay'я это **обычный IP-multicast пакет**. Underlay про VXLAN и VNI ничего не знает. Он просто видит «пакет с dst=239.1.1.1 летит в меня».

---

## Phase 3 — пакет попадает на R_core (first-hop router и RP)

R_core получает пакет с `src=10.0.0.1, dst=239.1.1.1`. Для него это **первый раз** когда он видит source `10.0.0.1` слать в эту группу.

```
  R_core — DR (Designated Router) для источника VTEP-A,
  потому что VTEP-A подключён к нему напрямую.

  Шаги:
  1. RPF-проверка: "если бы я слал unicast'ом к 10.0.0.1, через какой
     интерфейс? Через интерфейс к VTEP-A. Пакет пришёл оттуда же.
     RPF OK."

  2. R_core — это DR для этого source. Должен сообщить RP о появлении
     источника через PIM Register.

  3. Но в нашем сетапе R_core И ЕСТЬ RP. Шаг Register → самому себе
     вырождается: R_core сразу понимает "у меня есть source для G,
     RPT у меня построен, OIL = [к VTEP-A, к VTEP-B]".

  4. Форвардит пакет по OIL, исключая интерфейс прихода:
     → к VTEP-B       ✅ лить
     → к VTEP-A       ❌ пакет оттуда пришёл, RPF у VTEP-A его дропнет
                         плюс PIM Prune логика
```

В реалистичном сетапе, где RP **не совпадает** с first-hop router'ом source'а:

```
  R_src ─── (PIM Register unicast'ом к RP с пакетом внутри) ─── RP
                RP декапсулирует и раздаёт по своему RPT OIL
```

После нескольких пакетов RP может инициировать **SPT switchover**: послать PIM Join(S, G) напрямую к source, чтобы следующие пакеты шли по прямому SPT без encap в Register. Но первый пакет всегда через Register. В нашей упрощённой топологии этот шаг «схлопывается» потому что R_core = RP.

---

## Phase 4 — пакет доходит до VTEP-B через underlay

В нашем сетапе это просто один hop от R_core к VTEP-B. В более сложном underlay пакет может пройти несколько роутеров, на каждом:
- RPF-проверка против source или RP (в зависимости от того по какому дереву — SPT или RPT),
- форвардинг по OIL.

В итоге VTEP-B видит на своём `eth0`:

```
  Ethernet кадр:
    dst MAC = 01:00:5e:01:01:01   ← NIC VTEP-B его примет, потому что
                                      kernel в Phase 0 запрограммировал
                                      NIC принимать multicast для 239.1.1.1
    EtherType = IPv4

  Внутри:
    outer IP  src=10.0.0.1  dst=239.1.1.1
    outer UDP dst=4789
    VXLAN VNI=10
    inner Eth src=host_A dst=ff:ff:ff:ff:ff:ff
    inner payload = ARP Request
```

---

## Phase 5 — VTEP-B декапсулирует

```
  kernel VXLAN-модуль:
  1. Распознаёт пакет (UDP dst=4789 → VXLAN)
  2. Читает VNI=10 → знает, это кадр для vxlan10
  3. Снимает outer Eth/IP/UDP/VXLAN заголовки
  4. Остаётся inner Eth-кадр — ИДЕНТИЧНЫЙ тому что host_A отправил
  5. Отдаёт его в vxlan10 интерфейс
  6. vxlan10 → мост br0 → флуд на все порты (это же BUM, dst=broadcast)
  7. → уходит на eth1 → host_B получает

  ПАРАЛЛЕЛЬНО vxlan10 учится:
    inner src MAC = host_A  ──┐
    outer src IP  = 10.0.0.1  ├──▶ запись в FDB:
                              ┘     "host_A's MAC → dst=10.0.0.1 (VTEP-A)"
```

Вот **тот самый момент** когда VTEP-B получает знание `inner MAC → remote VTEP-IP` — через flood-and-learn, ровно как описано в [[vtep-mac-learning-problem]]. IP-multicast в underlay закрыл механику «как ARP долетел до всех VTEP'ов», а обучение MAC'ам — классическое bridge-поведение поверх этого.

---

## Phase 6 — host_B отвечает, и дальше unicast

```
  host_B видит ARP Request, отвечает:

    ARP Reply: "30.1.1.2 это я, MAC = dd:ee:ff:00:00:02"
      src MAC = dd:ee:ff:00:00:02
      dst MAC = aa:bb:cc:00:00:01 (host_A — unicast!)

  → VTEP-B: br0 → vxlan10
  → vxlan10 смотрит в FDB:
      dst MAC aa:bb:cc:00:00:01 → dst=10.0.0.1 (запомнил секунду назад!)
  → encap UNICAST (не multicast!):
      outer IP: src=10.0.0.2, dst=10.0.0.1
      VNI=10
      inner: ARP Reply
  → underlay просто unicast-роутит → VTEP-A

  PIM и multicast FIB в этом обратном пути НЕ участвуют
  — это обычный unicast IP-пакет.
```

Важный итог: **как только VTEP'ы «выучили» друг друга через один BUM-флуд, дальше они говорят между собой обычным unicast-ом**. Multicast-инфраструктура ([[igmp|IGMP]] + [[pim|PIM]]) работает только:
1. Для самого первого (и редких последующих) BUM-пакетов.
2. Для broadcast-событий: [[arp|ARP]] нового хоста, [[dhcp|DHCP]], старые протоколы discovery.
3. Для unknown unicast — когда VTEP забыл/ещё не знает MAC.

---

## Что где участвовало — резюме

| Роль | Кто | Когда |
|---|---|---|
| Подписка VTEP'а на группу underlay'я | [[igmp\|IGMP]] в ядре VTEP'а → first-hop router | При `ip link add` (boot time) |
| Построение RPT через underlay | [[pim\|PIM-SM]] Join от first-hop routers к RP | После того как IGMP Report дошёл |
| Доставка первого BUM | PIM Register (R_src → RP) + forwarding по RPT OIL | Первый пакет от source |
| Программирование NIC на multicast MAC | Ядро Linux при subscribe | При `ip link add` (boot time) |
| Обучение `inner MAC → remote VTEP IP` | Flood-and-learn на VTEP, по inner src MAC + outer src IP | При получении BUM-кадра |
| Последующий unicast VTEP↔VTEP | Обычный underlay unicast-роутинг | Сразу после обучения |

И ключевая мораль: **мультикаст в underlay'е работает на знакомство VTEP'ов, дальше трафик переходит на unicast**. Это делает цену multicast-режима относительно приемлемой — постоянно фладить BUM по дереву нужно не так много, после прогрева FDB основная масса трафика становится обычным unicast.
