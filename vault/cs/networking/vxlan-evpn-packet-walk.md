---
title: VXLAN EVPN — Packet Walk
tags:
  - concept
  - networking
  - overlay
  - cheatsheet
date: 2026-04-18
---

# VXLAN EVPN — Packet Walk

moc: [[networking-moc]]
back: [[vxlan-modes]]
next:
- [[bgp]]
- [[loopback]]
- [[vxlan-multicast-packet-walk]]
- [[vtep-mac-learning-problem]]
- [[bum-traffic]]
- [[vxlan]]
- [[arp]]

---

```
                      _wil-1  (RR)
                    lo: 1.1.1.1
                   ╱      │      ╲
                 e0       e1       e2
                ╱         │         ╲
          _wil-2        _wil-3       _wil-4        ← leaf'ы = VTEP'ы
        lo: 1.1.1.2   lo: 1.1.1.3  lo: 1.1.1.4
          │              │              │
        host-1         host-2         host-3
       20.1.1.1       20.1.1.2       20.1.1.3

  underlay: OSPF (для IP-связности loopback'ов)
  overlay:  VXLAN VNI=10, управляемый через BGP EVPN
  все в одной AS (iBGP), _wil-1 = Route Reflector
```

**TL;DR:** Практическая трасса ping'а от `host-1` к `host-2` через VXLAN в EVPN-режиме (BADASS P3). Показывает **Phase 0** — OSPF строит underlay, BGP-сессии поднимаются через [[loopback|loopback'и]], Type 3 формирует BUM-список без IGMP/PIM. **Phase 1** — host появляется → BGP Type 2 автоматически раздаёт MAC всем VTEP'ам **до любого трафика**. **Phase 2** — ARP идёт head-end replication (N unicast копий, без multicast), ответ — сразу unicast по FDB из BGP. **ARP suppression** — VTEP отвечает на ARP сам, BUM в underlay не летит вообще. Ср. с [[vxlan-multicast-packet-walk|multicast-режимом]].

## Phase 0 — что происходит ДО появления хостов

### 0a. OSPF строит underlay

OSPF стартует на всех роутерах, обменивается link-state. Результат: каждый роутер знает маршрут до [[loopback|loopback'ов]] всех остальных:

```
  на _wil-4:
    1.1.1.1/32 via 10.1.1.9 (e2)    ← к RR
    1.1.1.2/32 via 10.1.1.9 (e2)    ← к _wil-2 через RR
    1.1.1.3/32 via 10.1.1.9 (e2)    ← к _wil-3 через RR
    1.1.1.4/32 — connected, lo      ← я сам
```

Это фундамент: без этого [[bgp|BGP]]-сессии не поднимутся (peering через [[loopback|loopback'и]]), и VXLAN-туннели не заработают (VTEP-адреса = loopback'и).

### 0b. BGP-сессии поднимаются

Каждый leaf пирится с RR через iBGP:

```
  _wil-2 ──┐
  _wil-3 ──┼── TCP:179 ──▶  _wil-1 (RR)
  _wil-4 ──┘

  На каждом leaf'е:
    router bgp 1
      neighbor 1.1.1.1 remote-as 1     ← iBGP к RR
      address-family l2vpn evpn
        neighbor 1.1.1.1 activate
        advertise-all-vni
```

Пиринг идёт **через [[loopback|loopback'и]]** (не через физические IP интерфейсов). [[loopback|Loopback]] стабилен — если один линк упал, но есть альтернативный маршрут, BGP-сессия не порвётся.

### 0c. VXLAN-интерфейсы — БЕЗ multicast group

```bash
# на каждом leaf'е:
ip link add vxlan10 type vxlan id 10 dstport 4789 local 1.1.1.X
#                                          НЕТ параметра "group"!
```

Ключевое отличие от P2: **нет `group 239.1.1.1`**. Подписки на multicast нет, [[igmp|IGMP]] Report не шлётся, [[pim|PIM]]-дерево не строится. VXLAN создаётся «немым» — сам по себе не знает куда слать [[bum-traffic|BUM]].

### 0d. BGP Type 3 — замена IGMP+PIM

Как только `advertise-all-vni` включён и VNI 10 существует, FRR автоматически генерирует BGP Type 3 маршрут:

```
  _wil-2 → RR:  Type 3: "VTEP 1.1.1.2 — член VNI 10, шлите мне BUM"
  _wil-3 → RR:  Type 3: "VTEP 1.1.1.3 — член VNI 10"
  _wil-4 → RR:  Type 3: "VTEP 1.1.1.4 — член VNI 10"

  RR отражает каждый Type 3 всем остальным.
```

Результат: каждый VTEP знает всех других VTEP'ов в VNI 10 — это **head-end replication list** (куда дублировать BUM).

```
  P2 multicast:                        P3 EVPN:
  ──────────────                       ─────────
  IGMP Report → PIM Join → RPT        BGP Type 3 → RR → все знают
  underlay multicast нужен             underlay multicast НЕ нужен
  membership через data plane          membership через control plane
```

**Состояние после Phase 0**: все VTEP'ы знают друг друга через BGP. Хостов нет, Type 2 маршрутов нет. Но BUM-список готов.

---

## Phase 1 — host-1 появляется за _wil-2

host-1 включается. Его NIC подключён к bridge `br0` на `_wil-2`.

```
  host-1 подключён к br0
    │
    ▼
  br0 видит src MAC = 62:b7:1f:a6:5a:34 на своём порту
    │
    ▼
  FRR на _wil-2 замечает новый MAC на bridge →
  автоматически генерирует BGP Type 2 UPDATE:

    Route Distinguisher: 1.1.1.2:2
    [2]:[0]:[48]:[62:b7:1f:a6:5a:34]
    next-hop: 1.1.1.2
    RT: 1:10  (Route Target → VNI 10)

    "MAC 62:b7:1f:a6:5a:34 за VTEP 1.1.1.2, VNI 10"
    │
    ▼
  Шлёт к RR (_wil-1) через BGP UPDATE
    │
    ▼
  RR отражает к _wil-3 и _wil-4
    │
    ▼
  _wil-3 и _wil-4 записывают в FDB:
    "MAC 62:b7:.. → dst 1.1.1.2"
```

**Никакого flooding'а не произошло.** _wil-3 и _wil-4 знают MAC хоста за _wil-2 **заранее**, чисто через [[control-plane-data-plane|control plane]].

```
  P2 multicast:                        P3 EVPN:
  ──────────────                       ─────────
  MAC узнаётся из BUM-трафика          MAC узнаётся из BGP Type 2
  (flood-and-learn)                    (до любого трафика)
```

---

## Phase 2 — host-1 пингует host-2 (20.1.1.1 → 20.1.1.2)

### 2a. ARP Request — BUM через head-end replication

host-1 не знает MAC host-2. Шлёт [[arp|ARP]] broadcast:

```
  host-1: ARP Request "who has 20.1.1.2?"
    dst MAC = ff:ff:ff:ff:ff:ff (broadcast)
    │
    ▼
  br0 на _wil-2 → vxlan10 (BUM → flood)
    │
    ▼
  vxlan10: "куда слать BUM?"
  BUM replication list (из Type 3 маршрутов):
    - 1.1.1.3 (_wil-3)
    - 1.1.1.4 (_wil-4)
    │
    ▼
  Head-end replication — ДВА ОБЫЧНЫХ UNICAST пакета:

  Копия 1:  outer src=1.1.1.2  dst=1.1.1.3  (к _wil-3)
            VNI=10, inner=ARP broadcast

  Копия 2:  outer src=1.1.1.2  dst=1.1.1.4  (к _wil-4)
            VNI=10, inner=ARP broadcast
```

Underlay видит **обычные unicast IP-пакеты**, не multicast. OSPF роутит их как обычно. Никакого [[pim|PIM]], никаких multicast FIB.

```
  P2 multicast:                        P3 EVPN:
  ──────────────                       ─────────
  BUM → outer dst = 239.1.1.1         BUM → N unicast копий
  underlay мультикастит (PIM+IGMP)     underlay обычный unicast
  одна копия, сеть размножает          VTEP сам делает N копий
```

### 2b. ARP Reply — unicast, VTEP знает куда из BGP

_wil-3 декапсулирует ARP, отдаёт в br0 → host-2 получает ARP request, отвечает:

```
  host-2: ARP Reply
    dst MAC = 62:b7:1f:a6:5a:34 (host-1 — unicast)
    │
    ▼
  br0 на _wil-3 → vxlan10
    │
    ▼
  vxlan10 смотрит FDB: "MAC 62:b7:.. → dst 1.1.1.2"
  Откуда? ← из BGP Type 2, пришло в Phase 1, до любого трафика!
    │
    ▼
  VXLAN encap unicast: outer dst=1.1.1.2 → OSPF роутит → _wil-2
    │
    ▼
  _wil-2 decap → br0 → host-1 получает ARP Reply
```

**Главное отличие от multicast**: в P2 _wil-3 узнал бы MAC host-1 из inner src MAC пришедшего ARP (flood-and-learn). В P3 — _wil-3 знает **ещё до того как ARP долетел** (из BGP Type 2).

### 2c. Дальше — ICMP unicast

Оба хоста знают MAC'и, VTEP'ы знают FDB через BGP. Ping — чистый unicast VXLAN, ни BUM, ни BGP в data path.

---

## ARP suppression — бонус EVPN

Если в Type 2 маршруте помимо MAC указан ещё и **IP**, VTEP может отвечать на [[arp|ARP]] **сам, не фладя вообще**:

```
  host-1: ARP "who has 20.1.1.2?"
    │
    ▼
  _wil-2 смотрит в EVPN-таблицу:
  "20.1.1.2 → MAC xx:xx, за VTEP 1.1.1.3"  ← из BGP Type 2 с IP
    │
    ▼
  _wil-2 ЛОКАЛЬНО генерирует ARP Reply
    │
    ▼
  host-1 получает ответ мгновенно

  BUM-пакет в underlay НЕ УЛЕТАЕТ ВООБЩЕ
```

ARP suppression — VTEP, зная MAC+IP из BGP, подменяет собой ARP-ответчика. [[bum-traffic|BUM]]-трафик сводится к минимуму.

---

## Сравнение: P2 multicast vs P3 EVPN

| | P2 Multicast | P3 EVPN |
|---|---|---|
| Underlay routing | static | **OSPF** |
| Как VTEP узнаёт членов VNI | IGMP + PIM → RPT | **BGP Type 3 → RR** |
| Как VTEP узнаёт remote MAC'и | flood-and-learn из BUM | **BGP Type 2 → RR** |
| BUM delivery | IP-multicast через underlay | **head-end replication (N unicast)** |
| ARP | всегда фладится | **может подавляться (ARP suppression)** |
| Детекция падения VTEP | FDB timeout (минуты) | **BGP withdraw (секунды)** |
| Multicast в underlay | обязателен (PIM, RP, IGMP) | **не нужен** |
| Политика / фильтрация | нет | **BGP route-maps, communities** |

## Связь с BADASS P3

- _wil-1 = RR, OSPF + BGP
- _wil-2, _wil-3, _wil-4 = leaf'ы / VTEP'ы, OSPF + BGP EVPN + VXLAN VNI=10
- host-1, host-2, host-3 = busybox за leaf'ами
- Ср. с P2 ([[vxlan-multicast-packet-walk]]) — тот же VNI, но control plane полностью переехал в BGP
