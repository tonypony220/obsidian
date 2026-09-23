---
title: macOS Network Diagnostics
tags: [networking, macos, cheatsheet, tool]
date: 2026-09-23
---

# macOS Network Diagnostics

moc: [[networking-moc]]
next:
- [[linux-routing-table]]
- [[arp]]
- [[icmp]]

---

```text
конфигурация → маршрут → сосед → порт/DNS → TCP + TLS + HTTP
 networksetup    route       arp/ping    nc/dig      curl
```

**TL;DR:** Сначала read-only командами проверяем конфигурацию и выбранный маршрут, затем активными пробами локализуем сбой. `ping` шлюза может инициировать [[arp|ARP]], но не исправляет некорректную конфигурацию сети.

## Команды

| Команда | Что делает | Может ли «подтолкнуть» сеть |
|---|---|---|
| `networksetup -getinfo Wi-Fi` | Читает IP, маску и шлюз Wi-Fi | Нет |
| `route -n get default` | Показывает выбранный маршрут по умолчанию | Нет |
| `route -n get 1.1.1.1` | Показывает, через какой интерфейс пойдёт конкретный адрес | Нет |
| `route -n get 192.168.2.254` | Проверяет путь к Experia | Нет |
| `route -n get -ifscope en0 1.1.1.1` | Показывает маршрут при принудительном использовании Wi-Fi | Нет |
| `networksetup -listallhardwareports` | Сопоставляет hardware ports и интерфейсы; в этом случае `en0` — Wi-Fi, `en8` — iPhone USB | Нет |
| `arp -n 192.168.2.254` | Читает соответствие IP → MAC из локального ARP-кэша | Нет |
| `ping -c 4 1.1.1.1` | Отправляет четыре [[icmp|ICMP]]-пакета без участия DNS | Да, создаёт трафик |
| `ping -c 4 192.168.2.254` | Проверяет локальный шлюз и при необходимости инициирует ARP | Да; наиболее вероятная команда, которая могла «подтолкнуть» связь |
| `nc -vz 192.168.2.254 80` | Проверяет TCP-порт панели по HTTP | Да; может обновить ARP |
| `nc -vz 192.168.2.254 443` | Проверяет TCP-порт панели по HTTPS | Да; может обновить ARP |
| `curl http://192.168.2.254` | Загружает страницу панели без авторизации | Да, создаёт локальный HTTP-трафик |
| `curl --interface en0 http://1.1.1.1` | Проверяет интернет через Wi-Fi в обход телефона | Да; проверка WAN без DNS |
| `dig @192.168.2.254 example.com` | Напрямую спрашивает DNS Experia об адресе сайта | Да; только DNS-запрос |
| `curl --interface en0 https://example.com` | Проверяет полный путь через Wi-Fi: DNS + TCP + TLS + HTTP | Да |

## Порядок диагностики

```bash
networksetup -getinfo Wi-Fi
networksetup -listallhardwareports

route -n get default
route -n get 1.1.1.1
route -n get 192.168.2.254
route -n get -ifscope en0 1.1.1.1

arp -n 192.168.2.254
ping -c 4 192.168.2.254
ping -c 4 1.1.1.1

nc -vz 192.168.2.254 80
nc -vz 192.168.2.254 443
curl http://192.168.2.254

curl --interface en0 http://1.1.1.1
dig @192.168.2.254 example.com
curl --interface en0 https://example.com
```

## Как читать результат

- Шлюз не пингуется → проблема между Mac и локальным роутером: интерфейс, Wi-Fi, адресация или ARP.
- `1.1.1.1` доступен, а доменное имя нет → проверять DNS.
- `curl http://1.1.1.1` отвечает ошибкой HTTP, но соединяется → IP-маршрут и TCP работают; конкретный HTTP-ответ здесь не важен.
- Запрос работает только с `--interface en0` → обычный [[linux-routing-table|выбор маршрута]] уводит трафик через другой интерфейс, например iPhone USB.
