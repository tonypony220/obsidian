---
title: Linux Bridge
tags:
  - concept
  - networking
  - linux
  - l2
date: 2026-04-12
---

# Linux Bridge

moc: [[networking-moc]]
back: [[linux-virtual-network-devices]]
next: [[veth-pair]] [[vxlan-bridge-setup]] [[bum-traffic]] [[arp]]

---

```
                Linux Bridge (br0)
              ┌────────────────────┐
   eth1     ──┤ port 1              │
   veth-A   ──┤ port 2              │  ← виртуальный L2-свитч
   vxlan10  ──┤ port 3              │     внутри ядра
              └────────────────────┘
              FDB (MAC-таблица):
                MAC aa:01 → port 1
                MAC bb:02 → port 2
                ...
```

**TL;DR:** Linux bridge — виртуальный L2-свитч в ядре. Соединяет несколько L2-интерфейсов (физические NIC, veth, VXLAN, VLAN-subif) в один сегмент. Учится MAC-адресам через flood-and-learn, ведёт собственную FDB. Базовый строительный блок для контейнерных сетей и L2-overlay.

## Что делает

Bridge ведёт **FDB** (Forwarding Database, она же MAC-таблица) — какой MAC за каким портом. На каждый входящий кадр:

1. Запоминает src MAC + входной порт ("MAC X виден за портом 2")
2. Смотрит dst MAC:
   - **известный unicast** → отправляет в конкретный порт
   - **broadcast/multicast** → флудит на все порты кроме входного
   - **unknown unicast** → флудит на все порты кроме входного

Это классическая логика L2-свитча, реализованная в ядре. Подробнее про flooding — [[bum-traffic]].

## Что можно подключить как порт

К bridge можно цеплять любые L2-интерфейсы:

- физический NIC (`eth1`)
- одну из сторон [[veth-pair]] (для контейнеров)
- VLAN-subinterface (`eth0.10`)
- [[vxlan|VXLAN-интерфейс]] (`vxlan10`) — bridge не знает, что за ним туннель, для него это просто ещё один порт

```bash
ip link add br0 type bridge
ip link set br0 up

ip link set eth1 master br0
ip link set veth-A master br0
ip link set vxlan10 master br0

# IP-адрес вешается на сам br0, если bridge должен иметь L3-присутствие
ip addr add 10.0.0.1/24 dev br0
```

После `master br0` интерфейс становится **портом** bridge: его собственный IP/маршруты больше не работают на L3, кадры проходят через bridge.

## Где используется

- **Docker** — создаёт `docker0` при установке. Контейнеры через [[veth-pair]] цепляются к нему — каждый контейнер имеет свой veth, "локальная" сторона висит в `docker0`.
- **Kubernetes (CNI bridge)** — те же принципы для подовых сетей.
- **VXLAN overlay** — bridge соединяет локальные хосты и VXLAN-туннель в один L2-сегмент. Это центральный кейс в BADASS P2 (см. [[vxlan-bridge-setup]]).
- **KVM/libvirt** — VM подключаются через bridge к физической сети.

## Команды для просмотра

```bash
ip link show master br0          # все порты bridge'а
bridge link show                 # альтернативно
bridge fdb show                  # MAC-таблица (FDB) всех bridge'ей
bridge fdb show br br0           # только записи br0
```

В выводе `bridge fdb show` каждая запись — пара "MAC → за каким портом" (для VXLAN-портов также remote VTEP IP, см. [[vxlan-modes]]).
