---
title: VXLAN Modes — Static, Multicast, EVPN
tags:
  - concept
  - networking
  - overlay
  - tradeoff
date: 2026-04-15
---

# VXLAN Modes

moc: [[networking-moc]]
back: [[vxlan]]
next:
- [[vxlan-multicast-packet-walk]]
- [[vtep-mac-learning-problem]]
- [[control-plane-data-plane]]
- [[bum-traffic]]
- [[ip-multicast]]
- [[igmp]]
- [[pim]]
- [[autonomous-system]]
- [[igp-vs-egp]]
- [[neighbor-liveness]]
- [[arp]]
- [[vni]]
- [[overlay-underlay]]
- [[linux-bridge]]
- [[vxlan-bridge-setup]]
- [[tenant]]

---

```
  ┌────────── как VTEP узнаёт куда слать ──────────┐
  │                                                │
  │  STATIC       → админ руками пишет FDB         │
  │                 "MAC X за VTEP 10.0.0.2"       │
  │                                                │
  │  MULTICAST    → BUM через IP-multicast group   │
  │                 underlay доставляет BUM всем   │
  │                 unicast MAC — через learning   │
  │                                                │
  │  EVPN (BGP)   → control plane раздаёт MAC-VTEP │
  │                 map; никакого learning         │
  └────────────────────────────────────────────────┘
```

**TL;DR:** Чтобы работать, VTEP должен знать (а) где находится MAC получателя для unicast, (б) кто члены [[vni|VNI]] для BUM-флуда. Три режима — это три ответа на вопрос «где [[control-plane-data-plane|control plane]]»: **static** (нет CP, человек руками), **multicast** (CP делегирован underlay через PIM/IGMP, data plane учится сам), **EVPN** (настоящий отдельный CP на BGP). BADASS P2 использует первые два, P3 — третий.

## Рамка: три режима = три ответа про control plane

Чтобы доставить кадр, VTEP должен знать две вещи: **(а)** для unicast — за каким удалённым VTEP'ом лежит MAC получателя (FDB); **(б)** для [[bum-traffic|BUM]] — кто члены VNI. Почему VTEP не может узнать это сам — отдельно в [[vtep-mac-learning-problem]]. Вопрос только в том, **откуда** эту информацию взять снаружи. Это вопрос про [[control-plane-data-plane|control plane]]:

```
  STATIC     : control plane НЕТ
               → карту держит админ в голове и в конфигах

  MULTICAST  : control plane ЕСТЬ, но ЧУЖОЙ
               → underlay (PIM + IGMP) решает куда доставить BUM
               → VTEP учится MAC'ам из самого data plane (flood-and-learn)

  EVPN       : control plane ЕСТЬ, ОТДЕЛЬНЫЙ и ПОЛНОЦЕННЫЙ
               → BGP заранее раздаёт "MAC X за VTEP Y, VNI Z"
               → data plane только пересылает, ничему не учится
```

Это не три независимые фичи, а эволюция: от ручного, к «пусть подсеть разберётся сама», к «у нас есть настоящий протокол распределения состояния». Каждый следующий шаг — шире масштаб и меньше зависимости от свойств underlay.

### Head-end replication

Термин используется в static и EVPN. Смысл: у VTEP'а нет multicast в underlay, поэтому **BUM-кадр копируется VTEP'ом в N unicast-пакетов** — по одному на каждого удалённого VTEP в VNI. Всю работу по размножению делает head (source VTEP), отсюда название. Противоположность — IP-multicast, где underlay сам размножает копии.

## Static mode — ручной FDB

Админ **вручную** прописывает записи в FDB каждого VTEP'а. Подходит для маленьких лабораторий и когда список VTEP'ов известен и статичен. BUM-доставка идёт **head-end replication** — source VTEP сам размножает копии.

### Настройка в Linux

```bash
# создать VTEP без multicast/remote — "немой" VXLAN
ip link add vxlan10 type vxlan \
    id 10 dstport 4789 dev eth0 local 10.0.0.1

# вручную добавить remote VTEP для flooding BUM
bridge fdb append 00:00:00:00:00:00 dev vxlan10 dst 10.0.0.2
bridge fdb append 00:00:00:00:00:00 dev vxlan10 dst 10.0.0.3

# можно прописать unicast-записи для конкретных MAC'ов
bridge fdb append aa:bb:cc:dd:ee:ff dev vxlan10 dst 10.0.0.2
```

`00:00:00:00:00:00` — специальный маркер "всё неизвестное флуди сюда" (head-end replication). Если таких записей несколько — VTEP делает copy для каждого.

### Свойства

- ✅ Underlay может быть любой L3 (не нужен multicast)
- ✅ Простая отладка — всё видно в `bridge fdb show`
- ❌ Не масштабируется — каждое изменение требует правок на всех VTEP'ах
- ❌ Нет автоматического обнаружения новых VTEP'ов

## Multicast mode — flood-and-learn через IP-multicast

Каждый [[vni|VNI]] привязан к группе [[ip-multicast|IP-multicast]] в underlay. VTEP'ы подписываются на неё через [[igmp|IGMP]] (хост↔first-hop router), underlay-роутеры строят дерево доставки через [[pim|PIM]] (роутер↔роутер). BUM-трафик отправляется в группу — underlay сам размножает копии всем VTEP'ам этой группы.

### Как это работает

```
  VTEP-A хочет фладить ARP-запрос в VNI 10
     │
     ▼
  encap в VXLAN, outer dst IP = 239.1.1.1 (multicast)
     │
     ▼
  underlay с PIM доставляет копии всем VTEP'ам,
  подписанным на 239.1.1.1
     │
     ▼
  VTEP-B, VTEP-C декапсулируют и отдают в локальный bridge
```

### Learning — как VTEP'ы узнают MAC-адреса

Unicast-часть работает по механике **flood-and-learn** — такой же как у обычного [[linux-bridge|bridge]], только фладится не через порты свитча, а через multicast в underlay. VTEP-B видит [[arp|ARP]]-запрос от Host-1, запоминает inner src MAC + outer src IP, дальше шлёт unicast напрямую. Детальный разбор механики — в [[bum-traffic]].

### Настройка в Linux

```bash
ip link add vxlan10 type vxlan \
    id 10 \
    dstport 4789 \
    dev eth0 \
    group 239.1.1.1              # multicast group для этого VNI
```

Underlay-роутеры должны быть настроены с [[pim|PIM]] и знать как роутить `239.1.1.1/32` между VTEP'ами. [[igmp|IGMP]]-Report при создании интерфейса Linux-ядро шлёт автоматически.

### Свойства

- ✅ Масштабируется — добавил нового VTEP'а и всё
- ✅ Автообнаружение через learning
- ❌ Underlay должен поддерживать multicast (PIM) — не всегда есть в L3-fabric
- ❌ Flood-and-learn = повышенный BUM-трафик при обучении
- ❌ Нет контроля политики (кто куда видит)

## EVPN mode — control plane через BGP

Самый современный подход. BGP с EVPN address family (MP-BGP AFI/SAFI 25/70) переносит **контрольную информацию** о MAC-адресах и VTEP'ах между роутерами. VTEP'ы не учатся из flood — они знают заранее. BGP здесь — это [[igp-vs-egp|EGP-класса]] протокол, работающий между [[autonomous-system|AS]] (или внутри одной AS как iBGP в ДЦ-фабрике).

### Как это работает

```
  Host-1 появляется за VTEP-A
     │
     ▼
  VTEP-A отправляет BGP Type 2 маршрут:
  "MAC Host-1 лежит за VTEP-A (next-hop 10.0.0.1), VNI 10"
     │
     ▼
  Route Reflector раздаёт всем BGP-пирам
     │
     ▼
  VTEP-B, VTEP-C сразу знают куда слать unicast к Host-1
  — БЕЗ flood, БЕЗ learning
```

Типы EVPN-маршрутов (часто используемые):

- **Type 2** (MAC/IP Advertisement) — "вот MAC такой-то за мной, опционально с IP"
- **Type 3** (Inclusive Multicast) — "я член VNI такого-то, шлите мне BUM head-end replication"

### Свойства

- ✅ Масштабируется на дата-центры (тысячи VTEP'ов)
- ✅ Underlay L3-only, multicast не нужен
- ✅ Политика через BGP — фильтры, route-maps, communities
- ✅ Быстрая сходимость — падение VTEP видно через BGP [[neighbor-liveness|keepalive]], Type 2 снимаются
- ✅ ARP suppression — VTEP может отвечать на [[arp|ARP]] локально из BGP-данных
- ❌ Сложнее — нужен BGP-стек, IGP под ним, Route Reflector

### Настройка в FRR (упрощённо)

```
router bgp 65001
 neighbor 10.0.0.250 remote-as 65001       # peering с Route Reflector
 !
 address-family l2vpn evpn
  neighbor 10.0.0.250 activate
  advertise-all-vni                        # анонсить все локальные VNI
 exit-address-family
```

## Сравнение

| | Static | Multicast | EVPN |
|---|---|---|---|
| Underlay требования | любая L3 | L3 + PIM/IGMP | любая L3 |
| Control plane | нет | нет | BGP |
| Обнаружение MAC | вручную или learning | flood-and-learn | BGP Type 2 |
| Обнаружение VTEP | вручную | автоматом (IGMP) | BGP Type 3 |
| Масштаб | маленький | средний | большой |
| Сложность | низкая | средняя | высокая |
| BUM-трафик | head-end replication | IP-multicast flooding | head-end replication (по Type 3) |
| Политика | нет | нет | через BGP |

## В BADASS

- **P2 static** — `bridge fdb append ... dst <remote_vtep>` на каждом роутере вручную
- **P2 multicast** — VXLAN с `group 239.1.1.1`, минимум PIM в underlay. Пошаговая трасса BUM-пакета через весь underlay с учётом IGMP+PIM — в [[vxlan-multicast-packet-walk]].
- **P3 EVPN** — BGP с `address-family l2vpn evpn`, Route Reflector, автоматическое обнаружение MAC'ов хостов через Type 2. Пошаговая трасса — [[vxlan-evpn-packet-walk]].

Это иллюстрация эволюции подхода: от ручного к control-plane-driven, с каждым шагом проще поддерживать и лучше масштабируется.
