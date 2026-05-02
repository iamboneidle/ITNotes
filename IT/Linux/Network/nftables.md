___
___
# Tags
#linux
___
# Содержание
- [[#1. Определение]]
- [[#2. Работа с nftables]]
- [[#3. Базовые команды nft]]
- [[#4. Примеры]]
- [[#5. Расширенное использование nftables]]
	- [[#5.1 Логирование]]
	- [[#5.2 Множества (Sets)]]
	- [[#5.3 Состояние соединений (Stateful)]]
	- [[#5.4 NAT и редирект]]
	- [[#5.5 Ограничение скорости (Rate limiting)]]
___
# 1. Определение
**nftables** — это подсистема ядра Linux, пришедшая на смену `iptables`. Она обеспечивает фильтрацию пакетов и классификацию трафика. 
*Главные отличия*: более простой синтаксис, отсутствие жестко заданных таблиц по умолчанию (вы создаете их сами) и более высокая производительность.
___
# 2. Работа с nftables
В отличие от `iptables`, в `nftables` нет встроенных таблиц. Вы сами создаете иерархию: **Таблица -> Цепочка -> Правило**.
1. **Семейства (Families):** определяет, какой трафик обрабатывается.
    - `ip` (IPv4), `ip6` (IPv6), `inet` (IPv4 + IPv6), `arp`, `bridge`.
2. **Таблицы (Tables):** контейнеры для цепочек.
3. **Цепочки (Chains):** привязываются к "хукам" (hooks) ядра (input, output, forward, prerouting, postrouting).
4. **Правила (Rules):** конкретные условия и действия.
___
# 3. Базовые команды nft
- **Просмотр текущих правил:** `nft list ruleset` — показать вообще всё.
- **Создание таблицы:** `nft add table inet my_table`
- **Создание базовой цепочки:** `nft add chain inet my_table my_input { type filter hook input priority 0 \; policy accept \; }`
- **Добавление правила:** `nft add rule inet my_table my_input ip saddr 192.168.0.100 drop`
- **Удаление:** Чтобы удалить правило, нужно сначала узнать его дескриптор (handle): `nft list table inet my_table -a` `nft delete rule inet my_table my_input handle 5`
- **Сохранение и загрузка:** `nft list ruleset > /etc/nftables.conf` `nft -f /etc/nftables.conf`
___
# 4. Примеры
**Блокировка IP-адреса**
`nft add rule inet my_table my_input ip saddr 192.168.0.100 drop`

**Открытие порта для входящего трафика**
`nft add rule inet my_table my_input tcp dport 80 accept`

**Настройка простого межсетевого экрана**
```shell
# Очистка текущих правил
nft flush ruleset

# Создаем таблицу и цепочки
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0 ; policy drop ; }
nft add chain inet filter forward { type filter hook forward priority 0 ; policy drop ; }
nft add chain inet filter output { type filter hook output priority 0 ; policy accept ; }

# Разрешаем Loopback и уже установленные соединения
nft add rule inet filter input iif lo accept
nft add rule inet filter input ct state established,related accept

# Открываем порты SSH (22), HTTP (80), HTTPS (443)
nft add rule inet filter input tcp dport { 22, 80, 443 } accept
```
___
# 5. Расширенное использование nftables
## 5.1 Логирование
Логирование в `nftables` работает быстрее и гибче. Можно добавить префикс и ограничить частоту записей: `nft add rule inet filter input tcp dport 22 log prefix "SSH Access: " accept`
## 5.2 Множества (Sets)
Это "киллер-фича" nftables. Вместо 100 правил для 100 IP-адресов, вы создаете одно множество: `nft add set inet filter blacklisted { type ipv4_addr \; }` `nft add element inet filter blacklisted { 10.0.0.1, 10.0.0.2 }` `nft add rule inet filter input ip saddr @blacklisted drop`
## 5.3 Состояние соединений (Stateful)
Проверка состояния через модуль `ct` (conntrack): `nft add rule inet filter input ct state established,related accept`
## 5.4 NAT и редирект
Сначала нужно создать таблицу с типом `nat`: `nft add table ip nat` `nft add chain ip nat prerouting { type nat hook prerouting priority -100 \; }` `nft add rule ip nat prerouting tcp dport 80 redirect to :8080`
## 5.5 Ограничение скорости (Rate limiting)
Замена модулю `limit`: `nft add rule inet filter input tcp dport 22 ct state new limit rate 3/minute accept`
___
