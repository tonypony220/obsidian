---
title: Docker Networking
tags: [concept, networking, linux, docker]
date: 2026-04-15
---

# Docker Networking

moc: [[networking-moc]]
back: [[linux-virtual-network-devices]]
next: [[veth-pair]] [[linux-bridge]] [[linux-routing-table]]

---

```
   ┌─────────────────────────────────────────────┐
   │  HOST (ядро Linux = роутер)                 │
   │                                             │
   │   eth0 (физ.)  192.168.1.10                 │
   │    │                                        │
   │    │  iptables MASQUERADE (NAT)             │
   │    │  net.ipv4.ip_forward = 1               │
   │    │                                        │
   │   docker0 (bridge + L3 gateway)             │
   │    │  172.17.0.1/16                         │
   │    ├──────────┬──────────┐                  │
   │   veth        veth       veth               │
   └────┼──────────┼──────────┼──────────────────┘
        │          │          │
      eth0       eth0       eth0
      .0.2       .0.3       .0.4
     [ctn1]     [ctn2]     [ctn3]
```

**TL;DR:** отдельного «роутера» у Docker нет — маршрутизирует ядро хоста. `docker0` это L2-bridge с IP-адресом (gateway), контейнеры подключены к нему через [[veth-pair]], выход наружу идёт через `ip_forward=1` + `iptables MASQUERADE`.

## Компоненты

- **`docker0`** — [[linux-bridge]] с IP `172.17.0.1/16`. L2-свитч для контейнеров + их default gateway на L3.
- **[[veth-pair]]** — один конец в namespace контейнера (виден как `eth0`), другой — в хосте, воткнут в `docker0`.
- **Network namespace** — у каждого контейнера свой: свои интерфейсы, своя таблица маршрутов, свой iptables.
- **`net.ipv4.ip_forward=1`** — sysctl, разрешает ядру форвардить транзитные пакеты. Без него контейнер не сможет выйти наружу.
- **`iptables`** — NAT и фильтрация (см. ниже).

## Поток пакета «контейнер → интернет»

```
ctn1 eth0 (172.17.0.2)
  │ src=172.17.0.2, dst=8.8.8.8
  ▼
veth-пара (L2)
  │
  ▼
docker0 (ядро хоста: dst=8.8.8.8 → ip route → default via eth0)
  │
  ▼
FORWARD chain (iptables решает: пропускать?)
  │
  ▼
POSTROUTING: MASQUERADE → src переписан на 192.168.1.10
  │
  ▼
eth0 → провайдер
```

Обратный пакет приходит с `dst=192.168.1.10`, ядро по conntrack помнит соответствие и переписывает обратно на `172.17.0.2`.

## Port publishing (`-p 8080:80`)

DNAT-правило в цепочке `PREROUTING`:

```
dst=хост:8080  →  переписать на  172.17.0.2:80
                 → FORWARD через docker0
                 → контейнер получает пакет на eth0:80
```

Это **не proxy-процесс** — чистые iptables-правила в ядре. Docker-daemon только их конфигурирует.

## Почему так сделано

В ядре Linux уже есть полноценный L3-стек, NAT, conntrack, фильтры, [[linux-routing-table|таблицы маршрутов]]. Docker не изобретает роутер — он **конфигурирует** ядро: создаёт bridge, veth, namespace'ы и правила iptables. Вся «магия» — несколько `ip`/`iptables`/`sysctl` команд, обёрнутых в удобный CLI.

## Что увидишь на хосте

```bash
ip link show docker0                # bridge
ip addr show docker0                # 172.17.0.1/16
ip link | grep veth                 # концы veth-пар
iptables -t nat -L -n               # MASQUERADE, DNAT-правила
sysctl net.ipv4.ip_forward          # 1
```

Внутри контейнера:

```bash
ip addr show eth0                   # 172.17.0.X/16
ip route                            # default via 172.17.0.1
```
