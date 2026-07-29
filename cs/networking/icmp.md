---
title: ICMP
tags: [concept, networking, l3, protocol]
date: 2026-04-16
---

# ICMP

moc: [[networking-moc]]
back: [[l2-vs-l3]]
next:
- [[arp]]
- [[ip-multicast]]

---

```
  ┌─────────────────────────────┐
  │ Ethernet (L2)               │
  ├─────────────────────────────┤
  │ IP (L3)  protocol = 1       │  ← ICMP, не TCP (6), не UDP (17)
  ├─────────────────────────────┤
  │ ICMP    type=8 (Echo Req)   │
  │         type=0 (Echo Reply) │
  │         type=11 (TTL exc)   │
  │         type=3  (Unreach)   │
  ├─────────────────────────────┤
  │ payload (timestamp + data)  │
  └─────────────────────────────┘
```

**TL;DR:** ICMP — служебный протокол IP, сидит **прямо в IP-пакете** (`protocol = 1`), без TCP/UDP и без портов. Используется для control-сообщений (TTL exceeded, unreachable, fragmentation needed) и диагностики (ping, traceroute). Формально L3, иногда называют «L3.5», потому что обслуживает сам IP.

## Где в стеке

ICMP — на одном уровне с TCP и UDP, но **без L4**. Identifier для верхнего слоя в IP-заголовке: `protocol = 1` (TCP=6, UDP=17). Поэтому у ICMP **нет портов** — есть только `type` и `code`.

## Зачем нужен — две роли

**Control** (роутеры сообщают об ошибках доставки):

- `Destination Unreachable` (type 3) — нет маршрута, порт закрыт, фрагментация запрещена
- `Time Exceeded` (type 11) — TTL=0, пакет дропнут
- `Fragmentation Needed` (type 3, code 4) — нужен PMTU discovery
- `Redirect` (type 5) — «используй другой gateway для этой подсети»

**Диагностика** (утилиты на базе ICMP):

- `Echo Request` (type 8) / `Echo Reply` (type 0) — ping
- `Time Exceeded` — основа traceroute (см. ниже)

Это часть IP-стека, а не отдельный сервис — поэтому ICMP может сообщать про сам IP-уровень, что TCP/UDP сделать не могут.

## Ping — как работает

```
  A                                 B
  │ IP src=A dst=B                  │
  │ proto=1, ICMP type=8            │
  │ payload = timestamp T0          │
  ├────────────────────────────────▶│
  │                                 │ ядро отвечает автоматически
  │                                 │ (без user-space процесса)
  │ IP src=B dst=A                  │
  │ proto=1, ICMP type=0            │
  │ payload = тот же timestamp T0   │
  │◀────────────────────────────────┤
  │                                 │
  │ RTT = now − T0                  │
```

Ключевое: **отвечает ядро**, не пользовательский процесс. Поэтому пингуется хост без открытых портов.

## Traceroute — как использует ICMP

Шлёт пакеты с увеличивающимся TTL (1, 2, 3, ...). Каждый роутер на пути уменьшает TTL и при TTL=0 шлёт обратно ICMP `Time Exceeded` (type 11) от своего IP. По обратным ICMP traceroute восстанавливает цепочку L3-хопов.

Сам пробный пакет может быть UDP (классика на Linux), TCP SYN (`tcp` mode, обходит файрволы), ICMP Echo (Windows `tracert`). Но **ответы** — всегда ICMP.

Поэтому в [[l2-vs-l3|traceroute видны только L3-хопы]] (роутеры), а L2-свитчи — нет: они TTL не уменьшают.

## Файрвол и ICMP

ICMP не имеет портов — поэтому правило «разрешить TCP 80/443, всё остальное запретить» **автоматически режет ping**. Нужно отдельное правило по типу протокола (`-p icmp` в iptables), а не по порту.

Часто блокируют **только Echo** (чтобы хост не отвечал на ping), но **разрешают** `Time Exceeded` и `Fragmentation Needed` — иначе сломается PMTU discovery и traceroute от этого хоста перестанет работать корректно.

## ICMPv6 — критичная часть IPv6

В IPv6 ICMP взял на себя ещё и роли ARP (через [[arp|NDP]] — Neighbor Discovery) и IGMP (через MLD — Multicast Listener Discovery). Так что ICMPv6 — это **обязательная** часть IPv6: если его блокировать целиком, сеть не будет работать вообще (нет резолва соседей, нет автоконфигурации SLAAC).

## Команды

```bash
ping -c 4 8.8.8.8                    # 4 echo request'а
ping -s 1500 -M do 8.8.8.8           # PMTU discovery: don't fragment
traceroute 8.8.8.8                   # UDP probes
traceroute -I 8.8.8.8                # ICMP Echo probes
traceroute -T 8.8.8.8                # TCP SYN probes (обходит файрволы)
mtr 8.8.8.8                          # traceroute + ping в реальном времени

tcpdump -i eth0 icmp                 # видеть весь ICMP-трафик
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT   # разрешить ping
```
