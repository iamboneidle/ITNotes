___
___
# Tags
#kubernetes
___
# Содержание
- [[#0. Подготовка]]
	- [[#0.1. Конфигурация машин]]
	- [[#0.2. Подготовка]]
		- [[#0.2.1. Проверка swap]]
		- [[#0.2.2. Установка пакетов]]
		- [[#0.2.3 Активация модулей]]
- [[#1. Инициализация master'а]]
	- [[#1.1. Получение ip-адреса]]
	- [[#1.2. Инициализация ноды]]
	- [[#1.3. Копирование kube-конфига]]
	- [[#1.4. Установка CNI-плагина]]
- [[#2. Добавление slave-нод]]
	- [[#2.1. Получение токена]]
	- [[#2.2. Добавление ноды]]
___
# 0. Подготовка
## 0.1. Конфигурация машин
Имеем в начале следующий состав ВМ:
- **master**
  Ubuntu jammy v22.04
  4 CPU, 8GB RAM
- **slave**
  Ubuntu jammy v22.04
  4 CPU, 8GB RAM
Данные машины находятся в одном кладе и видят друг друга.
## 0.2. Подготовка
Данный список настроек следует выполнить как на **master**, так и на **slave**.
### 0.2.1. Проверка swap
SWAP нужно выключать (хотя k8s в последних версиях стал с ним работать более ли менее) для того, чтобы избегать флуктуаций различных в работе с памятью, чтобы k8s мог нормально мониториь нагрузку и эффективно с ней работать.
#### Проверка:
```shell
swapon --show
```
#### Отключение:
```shell
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```
### 0.2.2. Установка пакетов
#### Установка базовых утилит:
```shell
sudo apt-get update -y
sudo apt-get install -y apt-transport-https ca-certificates curl
```
#### Подключение k8s репы:
```shell
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
#### Установка k8s-компонентов
```shell
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl containerd
sudo apt-mark hold kubelet kubeadm kubectl
```
### 0.2.3 Активация модулей
#### Активация специфичных модулей
- **`br_netfilter`** — позволяет iptables видеть трафик который проходит через сетевые мосты (bridge).
  Kubernetes создаёт виртуальные мосты между подами. Без этого модуля iptables не сможет применять сетевые политики к трафику между подами — пакеты будут проходить мимо firewall незамеченными.
- **`overlay`** — файловая система overlayfs. Это то как Docker/containerd хранит слои контейнеров.
  Каждый образ контейнера состоит из слоёв (базовый ubuntu + твой код + зависимости). Overlay объединяет эти слои в одну файловую систему которую видит контейнер. Без него контейнеры не смогут запуститься.
```shell
sudo -i
modprobe br_netfilter
modprobe overlay
```
#### Настройка сети
- **`net.ipv4.ip_forward=1`** — включает IP форвардинг.
  По умолчанию Linux принимает пакеты только адресованные **себе** и отбрасывает всё остальное. IP форвардинг говорит ядру — если пакет пришёл не для меня, не выбрасывай его, а **перенаправь дальше**. Без этого нода не сможет маршрутизировать трафик между подами и наружу.
- **`net.bridge.bridge-nf-call-iptables=1`** — заставляет трафик через сетевые мосты проходить через iptables.
  Помнишь `br_netfilter` выше? Этот параметр его **активирует**. Модуль загружен, но без этой настройки он не работает. Говорит ядру: весь трафик через bridge — прогони через iptables для обработки сетевых политик.
- **`sysctl -p /etc/sysctl.conf`** — применяет настройки прямо сейчас без перезагрузки.
```shell
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables=1" >> /etc/sysctl.conf
sysctl -p /etc/sysctl.conf
```
#### Настройка cgroup driver
**cgroup (control groups)** — механизм ядра Linux который **ограничивает ресурсы** для процессов.
**В чём проблема без этой настройки:**
Есть два драйвера управления cgroup — `cgroupfs` и `systemd`.

- kubelet по умолчанию использует `systemd`
- containerd по умолчанию использует `cgroupfs`
Если они используют **разные драйверы** — это конфликт. Два менеджера пытаются управлять одними и теми же ресурсами по-разному. Kubelet будет нестабильным, ноды могут падать.
```shell
sudo mkdir /etc/containerd/
sudo vim /etc/containerd/config.toml
```
```ini
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
   [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```
```shell
sudo systemctl restart containerd
```
Говорит containerd — используй systemd для управления cgroup. Теперь kubelet и containerd говорят на одном языке.
___
# 1. Инициализация master'а
## 1.1. Получение ip-адреса:
```shell
ip a
```
## 1.2. Инициализация ноды
```shell
sudo kubeadm init \
  --apiserver-advertise-address=XXX.XX.XX.XX \ # ip-биндинг
  --pod-network-cidr=10.244.0.0/16 \ # подсеть для подов (default)
  --apiserver-cert-extra-sans=XXX.XX.XX.XX # для доступа извне
```
Если че пошло не так:
```shell
sudo kubeadm reset -f
```
## 1.3. Копирование kube-конфига
Из под обычного пользователя
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## 1.4. Установка CNI-плагина
Из под обычного пользователя
```shell
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
___
# 2. Добавление slave-нод
## 2.1. Получение токена
Выполняется на master-ноде
#### Генерация
```shell
sudo kubeadm token generate
```
#### Создание и получение команды
```shell
sudo kubeadm token create <token> --print-join-command --ttl=0
```
Итог:
```
kubeadm join XXX.XX.XX.XX:6443 --token xxxxxx.1yyyy1y4y7y6yy3y --discovery-token-ca-cert-hash sha256:ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
```
## 2.2. Добавление ноды
Далее полученную команду вводим на slave-ноде
```shell
sudo kubeadm join XXX.XX.XX.XX:6443 --token xxxxxx.1yyyy1y4y7y6yy3y --discovery-token-ca-cert-hash sha256:ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
```
#### Готово!
___
