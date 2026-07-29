---
title: ARP
tags: [concept, networking, l2, protocol]
date: 2026-04-15
---

# ARP

moc: [[networking-moc]]
back: [[l2-vs-l3]]
next: [[bum-traffic]] [[linux-bridge]] [[vxlan]]

---

```
  A (10.0.0.10)                              B (10.0.0.20)
        │
        │ "мне нужен MAC для 10.0.0.20"
        ▼
  ┌──────────────────────────────┐
  │ ARP request (L2 broadcast)   │ ──▶ все в сегменте
  │ dst MAC: ff:ff:ff:ff:ff:ff   │
  │ "who has 10.0.0.20?"         │
  └──────────────────────────────┘
                                             │
                                             ▼
                                  ┌──────────────────────┐
                                  │ ARP reply (unicast)  │
                         ◀────────│ "10.0.0.20 is at     │
                                  │  bb:bb:bb:bb:bb:bb"  │
                                  └──────────────────────┘
        │
        ▼
  A кэширует в ARP-таблице: 10.0.0.20 → bb:bb:bb:bb:bb:bb
  дальше шлёт Ethernet-кадры с dst MAC = bb:..
```

**TL;DR:** ARP — L2-протокол, который отвечает на вопрос «какой MAC у IP X?». Это **мост между L3 и L2**: IP-стек знает IP, но чтобы положить кадр в провод — нужен MAC. ARP работает через L2-broadcast **только внутри одного сегмента** и не ходит через роутеры.

## Зачем нужен

IP-стек выбирает next-hop как IP (см. [[l2-vs-l3]], [[linux-routing-table]]). Но L2 адресует **по MAC**. Чтобы отдать кадр соседу по сегменту, нужно знать его MAC. Вариантов найти MAC по IP два:

- спросить у всех сразу — **ARP**
- знать заранее из control plane — так делает EVPN в [[vxlan-modes|VXLAN]]

В обычных сетях control plane для L2 нет — только ARP.

## Без ARP L3 не работает (на Ethernet)

IP-стек умеет выбрать next-hop как IP, но **не умеет** положить кадр в Ethernet — для этого нужен MAC. Без ARP IP-связность через Ethernet физически невозможна.

Проверить просто: `ip neigh flush all` → следующий пакет упрётся в ARP request. Если ответ заблокировать (например, `iptables -A INPUT -p arp -j DROP` на соседе) — связь пропадёт сразу после истечения текущих записей в кэше.

Исключения, когда ARP не нужен:

- **Статический ARP** (`ip neigh add ...`) — MAC прописан вручную, запрос не шлётся
- **Point-to-point** (PPP, SLIP, GRE без Ethernet-инкапсуляции) — нет broadcast-домена, единственный сосед известен заранее
- **IPv6** — заменён на NDP (другой протокол, та же роль; см. ниже)
- **Loopback, unix-socket** — L2 нет вообще
- **EVPN ARP suppression** — control plane знает MAC заранее, локальный VTEP отвечает на ARP сам, не флудя в underlay (но это всё ещё ARP в overlay — просто без broadcast)

Вывод: ARP — не оптимизация, а обязательное звено. На Ethernet **L3 без ARP не работает**.

## Как работает

1. A хочет послать пакет на IP X (X — **в моём сегменте**, определяется через `ip route` с `scope link`).
2. A смотрит в локальный ARP-кэш (`ip neigh show`). Если есть — сразу использует MAC.
3. Если нет — шлёт ARP request: L2-кадр с `dst MAC = ff:ff:ff:ff:ff:ff` (broadcast), payload = «кто такой IP X? ответь мне».
4. Свитчи флудят broadcast на все порты сегмента. Все хосты получают, но отвечает только тот у кого IP = X.
5. Ответ приходит unicast'ом (ARP reply содержит MAC-адрес отправителя запроса — адресовать ответ можно точечно).
6. A кэширует `IP X → MAC Y`. TTL ~минуты (в Linux — `/proc/sys/net/ipv4/neigh/default/gc_stale_time`).

## Ограничения — ARP не ходит через роутеры

Broadcast ограничен **одним L2-сегментом**. ARP request **не уходит** за пределы broadcast-домена. Это значит:

- ARP'ить можно только соседа по сегменту (обычно — в той же IP-подсети)
- Для IP за роутером ARP'ится **роутер** (next-hop), а не конечный получатель
- В итоге кадр уходит с `dst MAC = gateway`, но `dst IP = конечный получатель`. Это и есть механизм роутинга hop-by-hop (см. [[l2-vs-l3]])

## Связь с BUM и flood-and-learn

ARP — **главный источник broadcast-трафика** в нормальной сети. Каждый request — broadcast на весь сегмент. Поэтому:

- Чем больше хостов в одном broadcast-домене, тем больше ARP-шума
- [[vlan|VLAN]] и [[vxlan|VXLAN]] ограничивают размер broadcast-домена — ARP летит только внутри сегмента
- [[bum-traffic|BUM]] обработка в свитче напрямую связана с ARP: если adresat уже «изучен» (MAC в таблице) — ARP reply летит unicast'ом, а не флудится

## ARP в VXLAN

ARP работает **в overlay как обычно**: контейнер шлёт ARP request на L2-интерфейс → кадр попадает в локальный bridge → bridge флудит на все порты, включая VXLAN-интерфейс → VTEP инкапсулирует кадр в UDP и раскидывает всем удалённым VTEP'ам того же VNI (см. [[vxlan-modes]]).

Оптимизация **ARP suppression** (в EVPN): control plane знает все MAC'и заранее, локальный VTEP отвечает на ARP вместо флуда через underlay.

## Команды

```bash
ip neigh show                         # ARP-таблица
ip neigh show dev eth0                # только для интерфейса
ip neigh flush all                    # очистить
ip neigh add 10.0.0.20 lladdr bb:... dev eth0  # статическая запись

arping -I eth0 10.0.0.20              # послать ARP вручную
tcpdump -i eth0 arp                   # посмотреть ARP-трафик
```

## IPv6 — не ARP, а NDP

В IPv6 ARP заменён на **NDP (Neighbor Discovery Protocol)**. Логика та же (найти L2-адрес по L3), но работает через ICMPv6 и использует multicast вместо broadcast. Кадры `Neighbor Solicitation` / `Neighbor Advertisement`. Смотреть — той же командой `ip neigh`.
