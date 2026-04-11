___
___
# Tags
#kubernetes 
___
# Содержание
- [[#0. Конфигурация кластера]]
	- [[#0.1. Машины]]
	- [[#0.2. Компоненты Kubernetes]]
	- [[#0.3. Проблемы связанные с разницей версий Debian]]
		- [[#0.3.1. OpenSSL util version mismatch]]
		- [[#0.3.2. Проблемы hostname]]
		- [[#0.3.3. Проблемы отсутствия root-пользователя]]
- [[#1. Настройка jumbox]]
	- [[#1.1. Скачивание утилит]]
	- [[#1.2. Скачивание бинарей]]
	- [[#1.3. Установка kubectl]]
- [[#2. Подготовка вычислительных ресурсов]]
	- [[#2.1. Файл с описанием машин]]
	- [[#2.2. Генерация и раскидывание ssh-ключей]]
		- [[#2.2.1. Копирование ssh-ключей]]
	- [[#2.3. Хостнеймы]]
		- [[#2.3.1. Создание hosts-файла]]
		- [[#2.3.2. Добавление hosts-файла на локальной машине]]
		- [[#2.3.3. Добавление hosts-файла на удаленных машинах]]
- [[#3. Подготовка CA и TLS сертификатов]]
	- [[#3.1. Certificate Authority]]
		- [[#3.1.1. Генерируем сертификат и приватный ключ]]
	- [[#3.2. Создаем сертификаты для клиента и сервера]]
	- [[#3.3. Распространяем сертификаты клиента и сервера]]
- [[#4. Генерация kube-конфигов для аутентификации]]
	- [[#4.1. Для worker nodes]]
	- [[#4.2. Для kube-proxy]]
	- [[#4.3. Для kube-controller-manager]]
	- [[#4.4. Для kube-scheduler]]
	- [[#4.5. Для admin]]
	- [[#4.6. Распространяем конфиги]]
- [[#5. Генерация шифрования данных и ключа]]
	- [[#5.1. Ключ шифрования]]
	- [[#5.2. Конфиг-файла шифрования]]
	- [[#5.3. Доставка ключа]]
- [[#6. Подъем etcd]]
	- [[#6.1. Копируем бинари etcd]]
	- [[#6.2. Настройка, старт]]
		- [[#6.2.1 Настройка]]
		- [[#6.2.2. Старт]]
		- [[#6.2.3. Проверка]]
- [[#7. Подъем Control Plane]]
	- [[#7.1. Копирование бинарей на сервер]]
	- [[#7.2. Подготовка Control Plane]]
		- [[#7.2.1. Установка бинарей Kubernetes Controller]]
		- [[#7.2.2. Конфигурация Controller Manager]]
		- [[#7.2.3. Конфигурация Scheduler]]
	- [[#7.3. Старт Controller Services]]
	- [[#7.4. Верификация]]
- [[#8. RBAC для Kubelet авторизации]]
	- [[#8.1. Верификация]]
- [[#9. Запуск Worker Nodes]]
	- [[#9.1. Предварительные шаги]]
	- [[#9.2. Подготовка worker node]]
		- [[#9.2.1. Установка пакетов]]
		- [[#9.2.2. Отключение swap]]
		- [[#9.2.3. Установка бинарников ноды]]
		- [[#9.2.4. Конфигурация CNI]]
		- [[#9.2.5. Конфигурация containerd]]
		- [[#9.2.6. Конфигурация Kubelet]]
		- [[#9.2.7. Конфигурация Kube-Proxy]]
		- [[#9.2.8. Старт Worker Service]]
- [[#10. Конфигурация удаленного доступа]]
- [[#11. Подготовка Pod Network Routes]]
	- [[#11.1. Routing Table]]
	- [[#11.2. Верификация]]
- [[#12. Smoke-тесты]]
	- [[#12.1. Data Encryption]]
	- [[#12.2. Deployments]]
	- [[#12.3. Port Forwarding]]
	- [[#12.4. Logs]]
	- [[#12.5. Exec]]
	- [[#12.6. Services]]
- [[#Литература]]
___
# 0. Конфигурация кластера
## 0.1. Машины
Имеем в начале следующий состав ВМ:
- **jumpbox**
  4 CPU, 8GB RAM
- **master**
  4 CPU, 8GB RAM
- **slave01**
  4 CPU, 8GB RAM
- **slvae02**
  4 CPU, 8GB RAM

Данные машины находятся в одном клауде и видят друг друга.
> [!note] 
> В оригинальном туториал используется дистрибутива `Debian 12`, у меня `Debian 11`. 

`hostnamectl` output:
```
   Static hostname: srv*-**********
         Icon name: computer-vm
           Chassis: vm
        Machine ID: ********************************
           Boot ID: ********************************
    Virtualization: kvm
  Operating System: Debian GNU/Linux 11 (bullseye)
            Kernel: Linux 5.10.0-28-cloud-amd64
      Architecture: x86-64
```
## 0.2. Компоненты Kubernetes
- [kubernetes](https://github.com/kubernetes/kubernetes) v1.32.x
- [containerd](https://github.com/containerd/containerd) v2.1.x
- [cni](https://github.com/containernetworking/cni) v1.6.x
- [etcd](https://github.com/etcd-io/etcd) v3.6.x
## 0.3. Проблемы связанные с разницей версий Debian
### 0.3.1. OpenSSL util version mismatch
В поставку Debian 11 входит `openssl v1`, а в Debian 12 `openssl v3`.
В качестве решения я просто собрал `openssl v3` из исходников:
Установка пакетов для сборки:
```shell
apt-get install -y build-essential checkinstall zlib1g-dev perl
```
Скачиваем исходники:
```shell
wget https://www.openssl.org/source/openssl-3.3.0.tar.gz
tar -xf openssl-3.3.0.tar.gz
cd openssl-3.3.0
```
Конфигурируем и собираем:
```shell
./config --prefix=/usr/local/openssl3 --openssldir=/usr/local/openssl3
make -j$(nproc)
make install
```
Если будет ошибка `error while loading shared libraries: libssl.so.3: cannot open shared object file: No such file or directory`, то нуно указать системе, где искать библиотеки:
```shell
/usr/local/openssl3/bin/openssl version
```
Создаем alias:
```shell
alias openssl3='/usr/local/openssl3/bin/openssl'
```
### 0.3.2. Проблемы hostname
> [!important] 
> Так как у не менял `hostname` и `fqdn` у машин, то постоянно приходилось править `server`, `node-0`, `node-1` `hostname`'ы и соответстующие им `fqdn`'ы. Тут надо быть внимательным, особенно в `ca.conf` все поправить и вставить правильно в команды при выдаче kube-конфигов для `kube-scheduler`,` kube-proxy` и остальных. Тем не менее в конспекте команды сделаны для мнемоник `server`, `node-0` и`node-1`.
### 0.3.3. Проблемы отсутствия root-пользователя
У меня отсутствовал `root`-пользователь для подключения к хостам, но было пользователь с правами `sudo`. Потому команды некоторые продублированы и для `root`-пользователя, и для пользователя с правами `sudo`.
___
# 1. Настройка jumbox
## 1.1. Скачивание утилит
```shell
{
  apt-get update
  apt-get -y install wget curl vim openssl git
}
```
## 1.2. Скачивание бинарей
```shell
git clone --depth 1 \
  https://github.com/kelseyhightower/kubernetes-the-hard-way.git
cd kubernetes-the-hard-way
```
В файле `downloads-amd64.txt` перечислены следующие зависимости:
```txt
https://dl.k8s.io/v1.32.3/bin/linux/amd64/kubectl
https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-apiserver
https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-controller-manager
https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-scheduler
https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-proxy
https://dl.k8s.io/v1.32.3/bin/linux/amd64/kubelet
https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.32.0/crictl-v1.32.0-linux-amd64.tar.gz
https://github.com/opencontainers/runc/releases/download/v1.3.0-rc.1/runc.amd64
https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-amd64-v1.6.2.tgz
https://github.com/containerd/containerd/releases/download/v2.1.0-beta.0/containerd-2.1.0-beta.0-linux-amd64.tar.gz
https://github.com/etcd-io/etcd/releases/download/v3.6.0-rc.3/etcd-v3.6.0-rc.3-linux-amd64.tar.gz
```
```shell
wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads-$(dpkg --print-architecture).txt
```
Разархивировать и красиво разложить
```shell
{
  ARCH=$(dpkg --print-architecture)
  mkdir -p downloads/{client,cni-plugins,controller,worker}
  tar -xvf downloads/crictl-v1.32.0-linux-${ARCH}.tar.gz \
    -C downloads/worker/
  tar -xvf downloads/containerd-2.1.0-beta.0-linux-${ARCH}.tar.gz \
    --strip-components 1 \
    -C downloads/worker/
  tar -xvf downloads/cni-plugins-linux-${ARCH}-v1.6.2.tgz \
    -C downloads/cni-plugins/
  tar -xvf downloads/etcd-v3.6.0-rc.3-linux-${ARCH}.tar.gz \
    -C downloads/ \
    --strip-components 1 \
    etcd-v3.6.0-rc.3-linux-${ARCH}/etcdctl \
    etcd-v3.6.0-rc.3-linux-${ARCH}/etcd
  mv downloads/{etcdctl,kubectl} downloads/client/
  mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} \
    downloads/controller/
  mv downloads/{kubelet,kube-proxy} downloads/worker/
  mv downloads/runc.${ARCH} downloads/worker/runc
}
rm -rf downloads/*gz
```
Накинуть бит исполнения
```shell
{
  chmod +x downloads/{client,cni-plugins,controller,worker}/*
}
```
## 1.3. Установка kubectl
```shell
{
  cp downloads/client/kubectl /usr/local/bin/
}
```
___
# 2. Подготовка вычислительных ресурсов
## 2.1. Файл с описанием машин
machines.txt:
```
IPV4_ADDRESS FQDN HOSTNAME POD_SUBNET
```
```txt
XXX.XXX.XXX.XXX server.kubernetes.local server
XXX.XXX.XXX.XXX node-0.kubernetes.local node-0 10.200.0.0/24
XXX.XXX.XXX.XXX node-1.kubernetes.local node-1 10.200.1.0/24
```
## 2.2. Генерация и раскидывание ssh-ключей
> [!note] 
> У меня отсутствует доступ к root-пользователю машин, однако у того пользователя, который мне доступен, есть права sudo. Потому дальнейшие команды будут выполняться не из по root, а из под пользователя с мнемоникой `NonRootUser`.
### 2.2.1. Копирование ssh-ключей
```shell
while read IP FQDN HOST SUBNET; do
  ssh-copy-id NonRootUser@${IP}
done < machines.txt
```
Теперь можно проверить, что ssh-подключение работает:
```shell
while read IP FQDN HOST SUBNET; do
  ssh -n NonRootUser@${IP} hostname
done < machines.txt
```
## 2.3. Хостнеймы
У меня hostname и fqdn у машин уже заданы, потому данный шаг мне не нужен.
Потому я разве что добавлю хосты в `/etc/hosts` на нодах и мастере, чтобы они были в курсе друг про друга (хотя это для меня не обязательно).
### 2.3.1. Создание hosts-файла:
```shell
echo "" > hosts
echo "# Kubernetes The Hard Way" >> hosts
```
```shell
while read IP FQDN HOST SUBNET; do
    ENTRY="${IP} ${FQDN} ${HOST}"
    echo $ENTRY >> hosts
done < machines.txt
```
### 2.3.2. Добавление hosts-файла на локальной машине
```shell
cat hosts >> /etc/hosts
```
Теперь можно подключаться по hostname:
```shell
for host in server node-0 node-1
   do ssh NonRootUser@${host} hostname
done
```
### 2.3.3. Добавление hosts-файла на удаленных машинах
```shell
while read IP FQDN HOST SUBNET; do
  scp hosts NonRootUser@${HOST}:~/
  ssh -n \
    NonRootUser@${HOST} "sudo tee -a /etc/hosts < hosts"
done < machines.txt
```
___
# 3. Подготовка CA и TLS сертификатов
Здесь будет подготовлена [PKI инфраструктура](https://en.wikipedia.org/wiki/Public_key_infrastructure) с помощью `openssl`, которая поднимает Certificate Authority. Сделаем TLS сертификаты для следующих компонентов:
- kube-apiserver;
- kube-controller-manager;
- kube-scheduler;
- kubelet;
- kube-proxy.
Команды следует исполнять с `jumpbox`.
## 3.1. Certificate Authority
В этом разделе мы настроим Центр Сертификации (CA), который будет использоваться для выпуска TLS-сертификатов для остальных компонентов Kubernetes. Настройка CA и генерация сертификатов через `openssl` может быть трудоёмкой, особенно если делаешь это впервые. Чтобы упростить задачу, в туториале уже есть готовый файл конфигурации `ca.conf`, в котором прописаны все параметры необходимые для генерации сертификатов для каждого компонента Kubernetes.
Вот, собственно, `ca.conf`:
```ini title=ca.conf fold
[req]
distinguished_name = req_distinguished_name
prompt             = no
x509_extensions    = ca_x509_extensions

[ca_x509_extensions]
basicConstraints = CA:TRUE
keyUsage         = cRLSign, keyCertSign

[req_distinguished_name]
C   = US
ST  = Washington
L   = Seattle
CN  = CA

[admin]
distinguished_name = admin_distinguished_name
prompt             = no
req_extensions     = default_req_extensions

[admin_distinguished_name]
CN = admin
O  = system:masters

# Service Accounts
#
# The Kubernetes Controller Manager leverages a key pair to generate
# and sign service account tokens as described in the
# [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/)
# documentation.

[service-accounts]
distinguished_name = service-accounts_distinguished_name
prompt             = no
req_extensions     = default_req_extensions

[service-accounts_distinguished_name]
CN = service-accounts

# Worker Nodes
#
# Kubernetes uses a [special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/)
# called Node Authorizer, that specifically authorizes API requests made
# by [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet).
# In order to be authorized by the Node Authorizer, Kubelets must use a credential
# that identifies them as being in the `system:nodes` group, with a username
# of `system:node:<nodeName>`.

[node-0]
distinguished_name = node-0_distinguished_name
prompt             = no
req_extensions     = node-0_req_extensions

[node-0_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Node-0 Certificate"
subjectAltName       = DNS:node-0, IP:127.0.0.1
subjectKeyIdentifier = hash

[node-0_distinguished_name]
CN = system:node:node-0
O  = system:nodes
C  = US
ST = Washington
L  = Seattle

[node-1]
distinguished_name = node-1_distinguished_name
prompt             = no
req_extensions     = node-1_req_extensions

[node-1_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Node-1 Certificate"
subjectAltName       = DNS:node-1, IP:127.0.0.1
subjectKeyIdentifier = hash

[node-1_distinguished_name]
CN = system:node:node-1
O  = system:nodes
C  = US
ST = Washington
L  = Seattle


# Kube Proxy Section
[kube-proxy]
distinguished_name = kube-proxy_distinguished_name
prompt             = no
req_extensions     = kube-proxy_req_extensions

[kube-proxy_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Kube Proxy Certificate"
subjectAltName       = DNS:kube-proxy, IP:127.0.0.1
subjectKeyIdentifier = hash

[kube-proxy_distinguished_name]
CN = system:kube-proxy
O  = system:node-proxier
C  = US
ST = Washington
L  = Seattle


# Controller Manager
[kube-controller-manager]
distinguished_name = kube-controller-manager_distinguished_name
prompt             = no
req_extensions     = kube-controller-manager_req_extensions

[kube-controller-manager_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Kube Controller Manager Certificate"
subjectAltName       = DNS:kube-controller-manager, IP:127.0.0.1
subjectKeyIdentifier = hash

[kube-controller-manager_distinguished_name]
CN = system:kube-controller-manager
O  = system:kube-controller-manager
C  = US
ST = Washington
L  = Seattle


# Scheduler
[kube-scheduler]
distinguished_name = kube-scheduler_distinguished_name
prompt             = no
req_extensions     = kube-scheduler_req_extensions

[kube-scheduler_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Kube Scheduler Certificate"
subjectAltName       = DNS:kube-scheduler, IP:127.0.0.1
subjectKeyIdentifier = hash

[kube-scheduler_distinguished_name]
CN = system:kube-scheduler
O  = system:system:kube-scheduler
C  = US
ST = Washington
L  = Seattle


# API Server
#
# The Kubernetes API server is automatically assigned the `kubernetes`
# internal dns name, which will be linked to the first IP address (`10.32.0.1`)
# from the address range (`10.32.0.0/24`) reserved for internal cluster
# services.

[kube-api-server]
distinguished_name = kube-api-server_distinguished_name
prompt             = no
req_extensions     = kube-api-server_req_extensions

[kube-api-server_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client, server
nsComment            = "Kube API Server Certificate"
subjectAltName       = @kube-api-server_alt_names
subjectKeyIdentifier = hash

[kube-api-server_alt_names]
IP.0  = 127.0.0.1
IP.1  = 10.32.0.1
DNS.0 = kubernetes
DNS.1 = kubernetes.default
DNS.2 = kubernetes.default.svc
DNS.3 = kubernetes.default.svc.cluster
DNS.4 = kubernetes.svc.cluster.local
DNS.5 = server.kubernetes.local
DNS.6 = api-server.kubernetes.local

[kube-api-server_distinguished_name]
CN = kubernetes
C  = US
ST = Washington
L  = Seattle


[default_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Admin Client Certificate"
subjectKeyIdentifier = hash
```
С данным файлом следует ознакомиться.
> [!attention] 
> Если `hostname` и `fqdn` серверов отличаются от тех, что предлагает и использует Kelsey Hightower по умолчанию в своих конспектах, то данный файл следует отредактировать.

> [!info] 
> Я использую команду `openssl3`, а не `openssl`, т.к. это alias на OpenSSL v3. См. [[#0.1.1. OpenSSL util version mismatch|П.0.1.1.]]
> 
### 3.1.1. Генерируем сертификат и приватный ключ
```shell
{
  openssl3 genrsa -out ca.key 4096
  openssl3 req -x509 -new -sha512 -noenc \
    -key ca.key -days 3653 \
    -config ca.conf \
    -out ca.crt
}
```
## 3.2. Создаем сертификаты для клиента и сервера
> [!attention] 
> Тут тоже надо подменить `node-0`&`node-1` по необходимости. 

```shell
certs=(
  "admin" "node-0" "node-1"
  "kube-proxy" "kube-scheduler"
  "kube-controller-manager"
  "kube-api-server"
  "service-accounts"
)
```
```shell
for i in ${certs[*]}; do
  openssl3 genrsa -out "${i}.key" 4096

  openssl3 req -new -key "${i}.key" -sha256 \
    -config "ca.conf" -section ${i} \
    -out "${i}.csr"

  openssl3 x509 -req -days 3653 -in "${i}.csr" \
    -copy_extensions copyall \
    -sha256 -CA "ca.crt" \
    -CAkey "ca.key" \
    -CAcreateserial \
    -out "${i}.crt"
done
```
## 3.3. Распространяем сертификаты клиента и сервера
Копируем сертификат на ноды
В общем случае это делается так:
```shell
for host in node-0 node-1; do
  ssh root@${host} mkdir /var/lib/kubelet/

  scp ca.crt root@${host}:/var/lib/kubelet/

  scp ${host}.crt \
    root@${host}:/var/lib/kubelet/kubelet.crt

  scp ${host}.key \
    root@${host}:/var/lib/kubelet/kubelet.key
done
```
В моем случае ввиду отсутствия root-пользователя так:
```shell title=NoRootUser fold
for host in node-0 node-1; do
  ssh NonRootUser@${host} "sudo mkdir -p /var/lib/kubelet/"
  
  scp ca.crt NonRootUser@${host}:~/
  scp ${host}.crt NonRootUser@${host}:~/
  scp ${host}.key NonRootUser@${host}:~/
  
  ssh NonRootUser@${host} "sudo mv ~/ca.crt /var/lib/kubelet/"
  ssh NonRootUser@${host} "sudo mv ~/${host}.crt /var/lib/kubelet/kubelet.crt"
  ssh NonRootUser@${host} "sudo mv ~/${host}.key /var/lib/kubelet/kubelet.key"
  
  ssh NonRootUser@${host} "sudo chmod 600 /var/lib/kubelet/kubelet.key"
  ssh NonRootUser@${host} "sudo chmod 644 /var/lib/kubelet/kubelet.crt /var/lib/kubelet/ca.crt"
done
```
На сервер в общем случае:
```shell
scp \
  ca.key ca.crt \
  kube-api-server.key kube-api-server.crt \
  service-accounts.key service-accounts.crt \
  root@server:~/
```
В моем случае ввиду отсутствия root-пользователя так:
```shell title=NoRootUser fold
{
  scp ca.key ca.crt kube-api-server.key kube-api-server.crt service-accounts.key service-accounts.crt NonRootUser@server:~/
  ssh NonRootUser@server "sudo mv ~/ca.key ~/ca.crt ~/kube-api-server.key ~/kube-api-server.crt ~/service-accounts.key ~/service-accounts.crt /root/"
  ssh NonRootUser@server "sudo chmod 600 /root/ca.key /root/kube-api-server.key /root/service-accounts.key"
  ssh NonRootUser@server "sudo chmod 644 /root/ca.crt /root/kube-api-server.crt /root/service-accounts.crt"
}
```
___
# 4. Генерация kube-конфигов для аутентификации
## 4.1. Для worker nodes
```shell fold
for host in node-0 node-1; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-credentials system:node:${host} \
    --client-certificate=${host}.crt \
    --client-key=${host}.key \
    --embed-certs=true \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${host} \
    --kubeconfig=${host}.kubeconfig

  kubectl config use-context default \
    --kubeconfig=${host}.kubeconfig
done
```
## 4.2. Для kube-proxy
```shell fold
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-proxy.kubeconfig
}
```
## 4.3. Для kube-controller-manager
```shell fold
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-controller-manager.kubeconfig
}
```
## 4.4. Для kube-scheduler
```shell fold
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-scheduler.kubeconfig
}
```
## 4.5. Для admin
```shell fold
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default \
    --kubeconfig=admin.kubeconfig
}
```
## 4.6. Распространяем конфиги
`kubelet` & `kube-proxy` идут на воркеры:
```shell
for host in node-0 node-1; do
  ssh root@${host} "mkdir -p /var/lib/{kube-proxy,kubelet}"

  scp kube-proxy.kubeconfig \
    root@${host}:/var/lib/kube-proxy/kubeconfig \

  scp ${host}.kubeconfig \
    root@${host}:/var/lib/kubelet/kubeconfig
done
```
`kube-controller-manager` & `kube-scheduler` на мастер:
```shell
scp admin.kubeconfig \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  root@server:~/
```
```shell title=NoRootUser(worker) fold
for host in node-0 node-1; do
  ssh NonRootUser@${host} "sudo mkdir -p /var/lib/{kube-proxy,kubelet}"
  
  scp kube-proxy.kubeconfig NonRootUser@${host}:~/
  scp ${host}.kubeconfig NonRootUser@${host}:~/
  
  ssh NonRootUser@${host} "sudo mv ~/kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig"
  ssh NonRootUser@${host} "sudo mv ~/${host}.kubeconfig /var/lib/kubelet/kubeconfig"
  
  ssh NonRootUser@${host} "sudo chmod 600 /var/lib/kube-proxy/kubeconfig /var/lib/kubelet/kubeconfig"
done
```
```shell title=NoRootUser(master) fold
{
  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig NonRootUser@server:~/
  ssh NonRootUser@server "sudo mv ~/admin.kubeconfig ~/kube-controller-manager.kubeconfig ~/kube-scheduler.kubeconfig /root/"
  ssh NonRootUser@server "sudo chmod 600 /root/admin.kubeconfig /root/kube-controller-manager.kubeconfig /root/kube-scheduler.kubeconfig"
}
```
___
# 5. Генерация шифрования данных и ключа
Kubernetes хранит большое количество данных о состоянии кластера, конфигурации приложений и секреты. Он позволяет шифровать данные.
## 5.1. Ключ шифрования
```shell
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```
## 5.2. Конфиг-файла шифрования
```shell
envsubst < configs/encryption-config.yaml \
  > encryption-config.yaml
```
## 5.3. Доставка ключа
```shell
scp encryption-config.yaml root@server:~/
```
```shell title=NoRootUser
{
	scp encryption-config.yaml NonRootUser@server:~/
	ssh NonRootUser@server "sudo mv ~/encryption-config.yaml /root/ && sudo chmod 600 /root/encryption-config.yaml"
}
```
___
# 6. Подъем etcd
## 6.1. Копируем бинари etcd
```shell
scp \
  downloads/controller/etcd \
  downloads/client/etcdctl \
  units/etcd.service \
  root@server:~/
```
```shell title=NoRootUser fold
{
  scp downloads/controller/etcd downloads/client/etcdctl units/etcd.service NonRootUser@server:~/
  ssh NonRootUser@server "sudo mv ~/etcd ~/etcdctl ~/etcd.service /root/"
}
```
## 6.2. Настройка, старт
Данные команды выполняются на `master` из под `root`.
### 6.2.1 Настройка
```shell
{
  mv etcd etcdctl /usr/local/bin/
}
```
```shell
{
  mkdir -p /etc/etcd /var/lib/etcd
  chmod 700 /var/lib/etcd
  cp ca.crt kube-api-server.key kube-api-server.crt \
    /etc/etcd/
}
```
### 6.2.2. Старт
```shell
mv etcd.service /etc/systemd/system/
```
```shell
{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd
}
```
### 6.2.3. Проверка
```shell
etcdctl member list
```
___
# 7. Подъем Control Plane
## 7.1. Копирование бинарей на сервер
```shell
scp \
  downloads/controller/kube-apiserver \
  downloads/controller/kube-controller-manager \
  downloads/controller/kube-scheduler \
  downloads/client/kubectl \
  units/kube-apiserver.service \
  units/kube-controller-manager.service \
  units/kube-scheduler.service \
  configs/kube-scheduler.yaml \
  configs/kube-apiserver-to-kubelet.yaml \
  root@server:~/
```
```shell title=NoRootUser fold
{
  scp \
    downloads/controller/kube-apiserver \
    downloads/controller/kube-controller-manager \
    downloads/controller/kube-scheduler \
    downloads/client/kubectl \
    units/kube-apiserver.service \
    units/kube-controller-manager.service \
    units/kube-scheduler.service \
    configs/kube-scheduler.yaml \
    configs/kube-apiserver-to-kubelet.yaml \
    NonRootUser@server:~/
  ssh NonRootUser@server "sudo mv ~/kube-apiserver ~/kube-controller-manager ~/kube-scheduler ~/kubectl ~/kube-apiserver.service ~/kube-controller-manager.service ~/kube-scheduler.service ~/kube-scheduler.yaml ~/kube-apiserver-to-kubelet.yaml /root/"
}
```
## 7.2. Подготовка Control Plane
Далее команды выполняются с `master` из под `root`.
Директория под kube-конфиг:
```shell
mkdir -p /etc/kubernetes/config
```
### 7.2.1. Установка бинарей Kubernetes Controller
```shell
{
  mv kube-apiserver \
    kube-controller-manager \
    kube-scheduler kubectl \
    /usr/local/bin/
}
```
```shell
{
  mkdir -p /var/lib/kubernetes/

  mv ca.crt ca.key \
    kube-api-server.key kube-api-server.crt \
    service-accounts.key service-accounts.crt \
    encryption-config.yaml \
    /var/lib/kubernetes/
}
```
```shell
mv kube-apiserver.service \
  /etc/systemd/system/kube-apiserver.service
```
### 7.2.2. Конфигурация Controller Manager
```shell
{
	mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
	mv kube-controller-manager.service /etc/systemd/system/
}
```
### 7.2.3. Конфигурация Scheduler
```shell
{
	mv kube-scheduler.kubeconfig /var/lib/kubernetes/
	mv kube-scheduler.yaml /etc/kubernetes/config/
	mv kube-scheduler.service /etc/systemd/system/
}
```
## 7.3. Старт Controller Services
```shell
{
  systemctl daemon-reload

  systemctl enable kube-apiserver \
    kube-controller-manager kube-scheduler

  systemctl start kube-apiserver \
    kube-controller-manager kube-scheduler
}
```
> [!note] 
> Надо дать секунд 10 на подъем Kube API Server. 
## 7.4. Верификация
```shell
kubectl cluster-info \
  --kubeconfig admin.kubeconfig
```
Output:
```
Kubernetes control plane is running at https://127.0.0.1:6443
```
___
# 8. RBAC для Kubelet авторизации
> RBAC - Role-Bases Access Control.

В этом разделе мы настроим RBAC-права, чтобы Kubernetes API Server мог обращаться к Kubelet API на каждой воркер-ноде. Доступ к Kubelet API необходим для получения метрик, логов и выполнения команд в подах.

В этом туториале у Kubelet флаг `--authorization-mode` выставлен в `Webhook`. В режиме Webhook авторизация проверяется через API [SubjectAccessReview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#checking-api-access).

Команды из этого раздела применяются ко всему кластеру и выполняются только на машине `server`.

Создаем `system:kube-apiserver-to-kubelet`  [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) с правами доступа к Kubelet API для решения общих задач управления подами:
```shell
kubectl apply -f kube-apiserver-to-kubelet.yaml \
  --kubeconfig admin.kubeconfig
```
## 8.1. Верификация
С `jumpbox` можно получить версию Kubernetes (обязательно с `fqdn`, а не `hostname`):
```shell
curl --cacert ca.crt \
  https://server.kubernetes.local:6443/version
```
Response:
```json
{
  "major": "1",
  "minor": "32",
  "gitVersion": "v1.32.3",
  "gitCommit": "32cc146f75aad04beaaa245a7157eb35063a9f99",
  "gitTreeState": "clean",
  "buildDate": "2025-03-11T19:52:21Z",
  "goVersion": "go1.23.6",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```
___
# 9. Запуск Worker Nodes
Тут будут запущены воркер-ноды, а также установлены компоненты:
- [runc](https://github.com/opencontainers/runc);
- [container networking plugins](https://github.com/containernetworking/cni);
- [containerd](https://github.com/containerd/containerd);
- [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet);
- [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).
## 9.1. Предварительные шаги
Шаги выполняются на `jumpbox`.
Копируем все бинарники и systemd-юнитфайлы на каждую ноду:
```shell
for HOST in node-0 node-1; do
  SUBNET=$(grep ${HOST} machines.txt | cut -d " " -f 4)
  sed "s|SUBNET|$SUBNET|g" \
    configs/10-bridge.conf > 10-bridge.conf

  sed "s|SUBNET|$SUBNET|g" \
    configs/kubelet-config.yaml > kubelet-config.yaml

  scp 10-bridge.conf kubelet-config.yaml \
  root@${HOST}:~/
done
```
```shell
for HOST in node-0 node-1; do
  scp \
    downloads/worker/* \
    downloads/client/kubectl \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    configs/kube-proxy-config.yaml \
    units/containerd.service \
    units/kubelet.service \
    units/kube-proxy.service \
    root@${HOST}:~/
done
```
```shell
for HOST in node-0 node-1; do
  scp \
    downloads/cni-plugins/* \
    root@${HOST}:~/cni-plugins/
done
```
```shell title=NoRootUser fold
for HOST in node-0 node-1; do
  SUBNET=$(grep ${HOST} machines.txt | cut -d " " -f 4)
  sed "s|SUBNET|$SUBNET|g" configs/10-bridge.conf > 10-bridge.conf
  sed "s|SUBNET|$SUBNET|g" configs/kubelet-config.yaml > kubelet-config.yaml
  
  scp 10-bridge.conf kubelet-config.yaml NonRootUser@${HOST}:~/
  ssh NonRootUser@${HOST} "sudo mv ~/10-bridge.conf ~/kubelet-config.yaml /root/"
done
```
```shell title=NoRootUser fold
for HOST in node-0 node-1; do
  scp \
    downloads/worker/* \
    downloads/client/kubectl \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    configs/kube-proxy-config.yaml \
    units/containerd.service \
    units/kubelet.service \
    units/kube-proxy.service \
    NonRootUser@${HOST}:~/
  ssh NonRootUser@${HOST} "sudo cp -r ~/* /root/"
done
```
```shell title=NoRootUser fold
for HOST in node-0 node-1; do
  ssh NonRootUser@${HOST} "mkdir -p ~/cni-plugins"
  scp downloads/cni-plugins/* NonRootUser@${HOST}:~/cni-plugins/
  ssh NonRootUser@${HOST} "sudo mkdir -p /root/cni-plugins && sudo mv ~/cni-plugins/* /root/cni-plugins/"
done
```
## 9.2. Подготовка worker node
Далее шагивы полняются на каждой `worker-node`.
### 9.2.1. Установка пакетов
```shell
{
  apt-get update
  apt-get -y install socat conntrack ipset kmod
}
```
### 9.2.2. Отключение swap
```
swapon --show
swapoff -a
```
### 9.2.3. Установка бинарников ноды
```shell
mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```
```shell
{
  mv crictl kube-proxy kubelet runc \
    /usr/local/bin/
  mv containerd containerd-shim-runc-v2 containerd-stress /bin/
  mv cni-plugins/* /opt/cni/bin/
}
```
### 9.2.4. Конфигурация CNI
```shell
mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/
```
Убедимся, что траффик идущий через CNI `bridge`, обрабатывается `iptables`.
```shell
{
  modprobe br-netfilter
  echo "br-netfilter" >> /etc/modules-load.d/modules.conf
}
```
```shell
{
  echo "net.bridge.bridge-nf-call-iptables = 1" \
    >> /etc/sysctl.d/kubernetes.conf
  echo "net.bridge.bridge-nf-call-ip6tables = 1" \
    >> /etc/sysctl.d/kubernetes.conf
  sysctl -p /etc/sysctl.d/kubernetes.conf
}
```
### 9.2.5. Конфигурация containerd
```shell
{
  mkdir -p /etc/containerd/
  mv containerd-config.toml /etc/containerd/config.toml
  mv containerd.service /etc/systemd/system/
}
```
### 9.2.6. Конфигурация Kubelet
```shell
{
  mv kubelet-config.yaml /var/lib/kubelet/
  mv kubelet.service /etc/systemd/system/
}
```
### 9.2.7. Конфигурация Kube-Proxy
```shell
{
  mv kube-proxy-config.yaml /var/lib/kube-proxy/
  mv kube-proxy.service /etc/systemd/system/
}
```
### 9.2.8. Старт Worker Service
```shell
{
  systemctl daemon-reload
  systemctl enable containerd kubelet kube-proxy
  systemctl start containerd kubelet kube-proxy
}
```
Тут `/var/lib/kubelet/kubeconfig` подменить `cluster.server`, если `fqdn` не стандартный.
Проверка запущенности `kubelet`:
```shell
systemctl is-active kubelet
```
___
# 10. Конфигурация удаленного доступа
Снова можно попинговать с jumpbox:
```shell
curl --cacert ca.crt \
  https://server.kubernetes.local:6443/version
```
Сделаем кубконфиг для админа:
```shell
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443/
  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true
  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin
  kubectl config use-context kubernetes-the-hard-way
}
```
___
# 11. Подготовка Pod Network Routes
Поды, запущенные на ноде, получают IP-адрес из диапазона Pod CIDR этой ноды. На данном этапе поды не могут общаться с подами на других нодах из-за отсутствия сетевых **маршрутов**.

В этом разделе мы создадим маршрут для каждой воркер-ноды, который связывает диапазон Pod CIDR ноды с её внутренним IP-адресом.

Существуют **и другие способы** реализации сетевой модели Kubernetes.
## 11.1. Routing Table
В этом разделе мы соберём информацию, необходимую для создания маршрутов в VPC-сети `kubernetes-the-hard-way`.
```
{
  SERVER_IP=$(grep server machines.txt | cut -d " " -f 1)
  NODE_0_IP=$(grep node-0 machines.txt | cut -d " " -f 1)
  NODE_0_SUBNET=$(grep node-0 machines.txt | cut -d " " -f 4)
  NODE_1_IP=$(grep node-1 machines.txt | cut -d " " -f 1)
  NODE_1_SUBNET=$(grep node-1 machines.txt | cut -d " " -f 4)
}
```
```shell
ssh root@server <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```
```shell
ssh root@node-0 <<EOF
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```
```shell
ssh root@node-1 <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
EOF
```
## 11.2. Верификация
```shell
ssh root@server ip route
```
```
default via XXX.XXX.XXX.XXX dev ens160 
10.200.0.0/24 via XXX.XXX.XXX.XXX dev ens160 
10.200.1.0/24 via XXX.XXX.XXX.XXX dev ens160 
XXX.XXX.XXX.0/24 dev ens160 proto kernel scope link src XXX.XXX.XXX.XXX 
```
```shell
ssh root@node-0 ip route
```
```
default via XXX.XXX.XXX.XXX dev ens160 
10.200.1.0/24 via XXX.XXX.XXX.XXX dev ens160 
XXX.XXX.XXX.0/24 dev ens160 proto kernel scope link src XXX.XXX.XXX.XXX 
```
```shell
ssh root@node-1 ip route
```
```
default via XXX.XXX.XXX.XXX dev ens160 
10.200.0.0/24 via XXX.XXX.XXX.XXX dev ens160 
XXX.XXX.XXX.0/24 dev ens160 proto kernel scope link src XXX.XXX.XXX.XXX 
```
___
# 12. Smoke-тесты
## 12.1. Data Encryption
```shell
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```
```shell
ssh root@server \
    'etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C'
```
```txt title=Output fold
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 4f 1b 80 d8 89 72 f4  |:v1:key1:O....r.|
00000050  60 8a 2c a0 76 1a e1 dc  98 d6 00 7a a4 2f f3 92  |`.,.v......z./..|
00000060  87 63 c9 22 f4 58 c8 27  b9 ff 2c 2e 1a b6 55 be  |.c.".X.'..,...U.|
00000070  d5 5c 4d 69 82 2f b7 e4  b3 b0 12 e1 58 c4 9c 77  |.\Mi./......X..w|
00000080  78 0c 1a 90 c9 c1 23 6c  73 8e 6e fd 8e 9c 3d 84  |x.....#ls.n...=.|
00000090  7d bf 69 81 ce c9 aa 38  be 3b dd 66 aa a3 33 27  |}.i....8.;.f..3'|
000000a0  df be 6d ac 1c 6d 8a 82  df b3 19 da 0f 93 94 1e  |..m..m..........|
000000b0  e0 7d 46 8d b5 14 d0 c5  97 e2 94 76 26 a8 cb 33  |.}F........v&..3|
000000c0  57 2a d0 27 a6 5a e1 76  a7 3f f0 b7 0a 7b ff 53  |W*.'.Z.v.?...{.S|
000000d0  cf c9 1a 18 5b 45 f8 b1  06 3b a9 45 02 76 23 61  |....[E...;.E.v#a|
000000e0  5e dc 86 cf 8e a4 d3 c9  5c 6a 6f e6 33 7b 5b 8f  |^.......\jo.3{[.|
000000f0  fb 8a 14 74 58 f9 49 2f  97 98 cc 5c d4 4a 10 1a  |...tX.I/...\.J..|
00000100  64 0a 79 21 68 a0 9e 7a  03 b7 19 e6 20 e4 1b ce  |d.y!h..z.... ...|
00000110  91 64 ce 90 d9 4f 86 ca  fb 45 2f d6 56 93 68 e1  |.d...O...E/.V.h.|
00000120  0b aa 8c a0 20 a6 97 fa  a1 de 07 6d 5b 4c 02 96  |.... ......m[L..|
00000130  31 70 20 83 16 f9 0a 22  5c 63 ad f1 ea 41 a7 1e  |1p ...."\c...A..|
00000140  29 1a d4 a4 e9 d7 0c 04  74 66 04 6d 73 d8 2e 3f  |).......tf.ms..?|
00000150  f0 b9 2f 77 bd 07 d7 7c  42 0a                    |../w...|B.|
0000015a
```
У ключа должен быть префикс `k8s:enc:aescbc:v1:key1`, что говорит о тома, что был использован провайдер `aescbc` для шифрования данных с ключом шифрования `key1`.
## 12.2. Deployments
Были обнаружены ошибки:
1. Ошибка в kube-конфиге kube-scheduler:
   `/var/lib/kubernetes/kube-scheduler.kubeconfig`. Заменил `fqdn`.
2. Ошибка в kube-конфиге kube-controller-manager:
   `/var/lib/kubernetes/kube-controller-manager.kubeconfig`. Заменил `fqdn`.

```shell
kubectl create deployment nginx \
  --image=nginx:latest
```
```
kubectl get pods -l app=nginx
```
Output:
```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-54c98b4f84-5twjw   1/1     Running   0          4m19s
```
## 12.3. Port Forwarding
```shell
POD_NAME=$(kubectl get pods -l app=nginx \
  -o jsonpath="{.items[0].metadata.name}")
```
```shell
kubectl port-forward $POD_NAME 8080:80
```
В новом терминале:
```shell
curl --head http://127.0.0.1:8080
```
Output:
```
HTTP/1.1 200 OK
Server: nginx/1.29.8
Date: Sat, 11 Apr 2026 17:36:15 GMT
Content-Type: text/html
Content-Length: 896
Last-Modified: Tue, 07 Apr 2026 11:37:12 GMT
Connection: keep-alive
ETag: "69d4ec68-380"
Accept-Ranges: bytes
```
## 12.4. Logs
```shell
kubectl logs $POD_NAME
```
В аутпуте должен быть след от нашего предыдущего `curl`:
```
127.0.0.1 - - [11/Apr/2026:17:36:15 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.74.0" "-"
```
## 12.5. Exec
```shell
kubectl exec -ti $POD_NAME -- nginx -v
```
```shell
nginx version: nginx/1.29.8
```
## 12.6. Services
Ошибки:
- Опять на обех нода неверный куб-конфиг у `kube-proxy`. Поменял `fqdn`.
  `/var/lib/kube-proxy/kubeconfig`.

В этом разделе мы проверим возможность открыть доступ к приложению через **Service**.
Откроем доступ к деплойменту `nginx` через сервис типа **NodePort**:
```
kubectl expose deployment nginx \
  --port 80 --type NodePort
```
Тип сервиса LoadBalancer использовать нельзя, так как кластер не настроен для работы с **облачным провайдером**. Настройка интеграции с облачным провайдером выходит за рамки этого туториала.
Получим номер порта, назначенного сервису `nginx`:
```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```
Получим имя ноды, на которой запущен под `nginx`:
```
NODE_NAME=$(kubectl get pods \
  -l app=nginx \
  -o jsonpath="{.items[0].spec.nodeName}")
```
Проверим доступность `nginx` по IP-адресу ноды и NodePort:
```
curl -I http://${NODE_NAME}:${NODE_PORT}
```
Ожидаемый результат:
```
HTTP/1.1 200 OK
Server: nginx/1.29.8
Date: Sat, 11 Apr 2026 17:46:28 GMT
Content-Type: text/html
Content-Length: 896
Last-Modified: Tue, 07 Apr 2026 11:37:12 GMT
Connection: keep-alive
ETag: "69d4ec68-380"
Accept-Ranges: bytes
```

Если все тесты прошли, то кластер жив.
___
# Литература
- [девопсим потихоньку](https://youtu.be/HqRKOz1UBXA?is=3s_i1ht4UNnVvEOV)
- [Kelsey Hightower — Kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
