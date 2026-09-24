---
title: veth pair
tags: [concept, networking, linux]
date: 2026-04-12
---

# veth pair

back: [[linux-virtual-network-devices]]
next: [[linux-bridge]] [[docker-networking]]

---

```
┌───────────┐                    ┌───────────┐
│ Container1│                    │ Container2│
│           │                    │           │
│   veth0 ●─┼────────────────────┼─● veth1   │
│           │  "виртуальный      │           │
└───────────┘   кабель"          └───────────┘
```

**veth** = virtual ethernet. Пара виртуальных Ethernet-интерфейсов, соединённых друг с другом напрямую. Работает на L2 (канальный уровень OSI) — как настоящий Ethernet-кабель, только программный.

## Как работает

Кадр (Ethernet frame) входит в `veth0` — выходит из `veth1`. Точно так же, как если бы ты воткнул реальный Ethernet-кабель между двумя машинами. Всегда создаётся парой — один конец без другого не имеет смысла.

## Типичное использование

```bash
# создать пару
ip link add veth0 type veth peer name veth1

# один конец — в namespace контейнера
ip link set veth1 netns container1

# второй — подключить к bridge
ip link set veth0 master br0
```

В Docker каждый контейнер получает свой конец veth-пары. Второй конец подключается к [[linux-bridge]] `docker0`.
