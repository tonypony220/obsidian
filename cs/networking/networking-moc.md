---
title: Networking MOC
tags:
  - moc
  - networking
date: 2026-04-12
---

# Networking MOC

## Фундамент

- [[l2-vs-l3]] — разделение слоёв: MAC меняется per-hop, IP end-to-end
- [[l2-segment]] — broadcast-домен: границы, что его определяет
- [[arp]] — мост между L2 и L3: «дай MAC по IP»
- [[icmp]] — служебный протокол IP: ping, traceroute, control-сообщения
- [[overlay-underlay]] — виртуальная сеть поверх физической, граница — VTEP
- [[tenant]] — изолированная логическая среда на общей инфре (multi-tenancy)
- [[bum-traffic]] — broadcast/unknown/multicast, почему L2 вообще флудит
- [[uplink-downlink]] — относительные термины роли порта в иерархии (к верху / к низу)
- [[interface-ip-assignment]] — когда интерфейсу нужен IP: endpoint vs transit, с примерами
- [[loopback]] — виртуальный интерфейс «всегда up»: localhost, router-id для BGP/OSPF, anycast

## API и протоколы

- [[rest-api]] — архитектурный стиль на HTTP-методах и ресурсах
- [[grpc]] — бинарный RPC-протокол на HTTP/2 + Protobuf
- [[api-gateway]] — единая точка входа для микросервисов

## Доставка контента

- [[cdn-caching]] — кэширование на edge-серверах CDN
- [[edge-vs-cdn]] — трейдофф между edge compute и CDN

## Маршрутизация (L3)

- [[l3-routing-moc]] — как пакеты находят путь: IP, таблицы маршрутов, протоколы
- [[bgp]] — протокол обмена маршрутами через TCP-сессию; MP-BGP, EVPN, Route Reflector

## Протоколы конфигурации

- [[dhcp]] — автоматическое получение IP, DORA, lease

## Linux virtual networking

- [[linux-virtual-network-devices]] — обзор: veth, bridge, tun/tap, vxlan
- [[veth-pair]] — виртуальный Ethernet-кабель (L2)
- [[linux-bridge]] — виртуальный свитч (L2)
- [[tun-tap]] — виртуальная сетевая карта (L2/L3)
- [[vxlan]] — L2 overlay-туннель через UDP
- [[docker-networking]] — как Docker собирает сеть из bridge + veth + iptables

## L2 сегментация и overlay

- [[vlan]] — логическое разделение одного свитча (802.1Q, 4094 сегмента)
- [[vxlan]] — L2 поверх L3 через UDP (VNI, VTEP, 16M сегментов)
- [[vxlan-bridge-setup]] — как собрать VXLAN + bridge на роутере, зачем bridge рядом с VTEP
- [[vtep-mac-learning-problem]] — почему VTEP сам не может узнать "за каким VTEP'ом чужой MAC"
- [[vxlan-modes]] — static / multicast / EVPN — как VTEP узнаёт куда слать
- [[vni]] — идентификатор L2-сегмента в VXLAN (24 бита, 16M сегментов)

## IP-multicast

- [[ip-multicast]] — доставка группе подписчиков через групповой IP; membership + distribution
- [[igmp]] — хост↔first-hop router: "я в группе G"; + IGMP snooping на свитчах
- [[pim]] — между роутерами: построение деревьев доставки; SM / DM / SSM
- [[vxlan-multicast-packet-walk]] — практическая трасса BUM-пакета через VXLAN multicast mode, Phase 0–6
- [[vxlan-evpn-packet-walk]] — практическая трасса через VXLAN EVPN mode: Type 2/3, head-end replication, ARP suppression
