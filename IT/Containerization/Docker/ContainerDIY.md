___
___
# Tags
#containerization
___
# Содержание
- [[#1. Теория]]
	- [[#1.1 Namespace]]
	- [[#1.2 Cgroups]]
- [[#2. Создание контейнера без Docker]]
- [[#Литература]]
___
# 1. Теория
> [!important] 
> Контейнеры в Linux держатся на двух механизмах ядра Linux - ***namespace*** и ***cgroups***. 

Оба этих механизма дают изоляцию и масштабируемость.
## 1.1 Namespace
***Namespace (пространства имен)*** позволяют изолировать ресурсы системы между процессами. С помощью них можно создавать новую изолированную систему, оставаясь фактически на хостовой.

Кратко о видах пространств имен, их всего **8**:
1. ***Mount*** - изоляция точек монтирования файловой системы. Позволяет установить свою иерархию файловой системы;
2. ***UTS*** - изоляция имени хоста. Позволяет для каждого контейнера указать свое хостовое имя;
3. ***PID*** - изоляция идентификаторов процессов. Позволяет создавать отдельное дерево процессов;
4. ***Network*** - изоляция сетевых интерфейсов, таблиц маршрутизации;
5. ***IPC*** - изоляция IPC (межпроцессные взаимодействия);
6. ***User*** - изоляция пользователей системы. Позволяет создавать отдельных пользователей для каждого контейнера, в том числе и root;
7. ***Cgroups*** - изоляция доступа к cgroup. Позволяет ограничивать ресурсы контейнера и предотвращает вмешательства других контейнеров;
8. ***Time*** - изоляция системного времени.
Для создания нового namespace в Linux существует команда `unshare`.
## 1.2 Cgroups
***Control groups*** - механизм ядра Linux, позволяющий управлять ресурсами процессов. С его помощью можно ограничить и изолировать использование CPU, памяти, сети, диска.
Существуют две версии cgroups - v1 и v2. В большинстве случаев используется v2, например в `systemd`. 

> [!info] 
> Основное отличие версий в построении дерева ограничений. В первой версии создавались узлы для каждого вида ограничений, а в них уже добавлялись группы. Во второй версии для каждой группы свой узел, внутри которого все необходимые ограничения. 

Визуализация деревьев разных версий:
```
#v1
/sys/fs/cgroup/
├── cpu
│   ├── group1/
│   │   ├── tasks
│   │   ├── cgroup.procs
│   │   ├── cpu.shares
│   │   └── ...
│   ├── group2/
│   │   ├── tasks
│   │   ├── cgroup.procs
│   │   ├── cpu.shares
│   │   └── ...
│   └── ...
├── memory
│   ├── group1/
│   │   ├── tasks
│   │   ├── cgroup.procs
│   │   ├── memory.limit_in_bytes
│   │   └── ...
│   ├── group2/
│   │   ├── tasks
│   │   ├── cgroup.procs
│   │   ├── memory.limit_in_bytes
│   │   └── ...
│   └── ...
└── ...

#v2
/sys/fs/cgroup/
├── group1/
│   ├── cgroup.procs
│   ├── cpu.max
│   ├── cpu.weight
│   ├── memory.current
│   ├── memory.max
│   └── ...
├── group2/
│   ├── cgroup.procs
│   ├── cpu.max
│   ├── cpu.weight
│   ├── memory.current
│   ├── memory.max
│   └── ...
└── ...
```
___
# 2. Создание контейнера без Docker
Для начала создадим структуру файловой системы контейнера, установим `busybox` в директорию `/bin` (`busybox` содержит в себе базовые утилиты типо `ls`):

```shell
# Создаем корневую директорию контейнера и переходим в нее
mkdir ~/container && cd ~/container
# Создаем основные системные директории и переходим в /bin
mkdir -p ./{proc,sys,dev,tmp,bin,root,etc} && cd bin
# Устанавливаем busybox
wget https://www.busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox
# Выдаем право на исполнение
chmod +x busybox
# Создаем симлинки для всех команд, которые есть в busybox 
./busybox --list | xargs -I {} ln -s busybox {}
# Возвращаемся в корневую директорию контейнера
cd ~/container
# Добавляем переменную PATH в файл 
/etc/profileecho 'export PATH=/bin' > ~/container/etc/profile
# Добавляем root-пользователя
echo "root:x:0:0:root:/root:/bin/sh" > ~/container/etc/passwd
echo "root:x:0:" > ~/container/etc/group
# Монтируем системные директории
# Монтируем устройства, используя уже существующие
sudo mount --bind /dev ~/container/dev
# Монтируем процессы
sudo mount -t proc none ~/container/proc
# Монтируем файловую систему sysfs
sudo mount -t sysfs none ~/container/sys
# Монтируем файловую систему tmpfs
sudo mount -t tmpfs none ~/container/tmp
```
Примечание:
Чтобы размонтировать есть команда:
```shell
sudo umount ~/container/{proc,sys,dev,tmp}  
```
Файловая система готова, создание изоляции:
```shell
unshare -f -p -m -n -i -u -U --map-root-user --mount-proc=./proc \    /bin/chroot ~/container /bin/sh -c "source /etc/profile && exec /bin/sh"
```
Что в команде:
- `-f` - fork. Создаем новый процесс для изоляции от родительского;
- `-p` - PID namespace;
- `-m` - mount namespace;
- `-n` - Network namespace;
- `-i` - IPC namespace;
- `-u` - UTS namespace;
- `-U` - User namespace;
- `--map-root-user` - маппинг uid и gid активного пользователя на root внутри контейнера;
- `-mount-proc` - монтируем proc внутри контейнера;
- `/bin/chroot ~/container` - меняем корневую директорию;
- `/bin/sh -c "source /etc/profile && exec /bin/sh"` - запускаем shell и исполняем команду, которая применит файл `/etc/profile` и запустит интерактивный shell.
Осталось ограничить ресурсы. Открываем новую сессию на хосте:
```shell
# Создаем новую группу. Для cgroups v2 директория автоматически будет настроена для работы с ресурсами
sudo mkdir /sys/fs/cgroup/my_container
# Записываем ограничение на 2 ядра процессора
echo "200000 100000" | sudo tee /sys/fs/cgroup/my_container/cpu.max
# Выделяем максимум 512MB памяти
echo 536870912 | sudo tee /sys/fs/cgroup/my_container/memory.max
```
Определяем PID контейнера:
```shell
ps aux | grep -E '/bin/sh$' 
```
Берем PID из второго столбца и добавляем в файл `cgroup.procs` :
```shell
echo <PID> | sudo tee /sys/fs/cgroup/my_container/cgroup.procs
```
Создана изолированная среда, настроены ограничения ресурсов. Настроим виртуальную сеть между контейнером и хостом:
```shell
# Создаем пару виртуальных интерфейсов
sudo ip link add veth-host type veth peer name veth-container
# Поднимаем интерфейс на хосте
sudo ip link set veth-host up
# Назначаем любой свободный адрес в сети на интерфейс хоста
# Например 192.168.1.123/24
sudo ip addr add 192.168.1.123/24 dev veth-host
# Перемещаем veth-container в пространство имен контейнера
# Здесь нужно указать PID контейнера, который использовали до этого
sudo ip link set veth-container netns <PID>
# Поднимаем интерфейс внутри контейнера
sudo nsenter --net=/proc/<PID>/ns/net ip link set veth-container up
# Назначаем любой свободный адрес в сети на интерфейс контейнера
# Например 192.168.1.124/24
sudo nsenter --net=/proc/<PID>/ns/net ip addr add 192.168.1.124/24 dev veth-container
# Настраиваем шлюз по умолчанию для маршрутизации трафика
sudo nsenter --net=/proc/<PID>/ns/net ip route add default via 192.168.1.123
```
Теперь необходимо настроить маршрутизацию:
```shell
# Разрешаем пересылку пакетов
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
# Добавляем правило NAT для маскарадинга для исходящих пакетов из сети 
# 192.168.1.0/24 через интерфейс который смотрит во внешнюю сеть, enp3s0.
# Маскарадинг маскирует пакеты исходящие их контейнера так, чтобы они выглядели,
# как пакеты отправленные с хоста
sudo iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o enp3s0 -j MASQUERADE
# Добавляем правило на разрешение форвардинга пакетов
sudo iptables -A FORWARD -s 192.168.1.0/24 -o enp3s0 -j ACCEPT
# Добавляем правило, разрешающее входящие пакеты
sudo iptables -A FORWARD -d 192.168.1.0/24 -m state --state RELATED,ESTABLISHED -j ACCEPT
```
___
# Литература
- [Хабр | net0pyr | Мой первый контейнер без Docker](https://habr.com/ru/articles/881428/)