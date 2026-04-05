___
___
# Tags
#kubernetes 
___
# Содержание
- [[#1. Устройство A-записей DNS]]
- [[#2. Обнаружение даже неготовых модулей]]
- [[#Литература]]
___
# 1. Устройство A-записей DNS
Для выполнения действий, связанных с DNS, можно использовать образ контейнера `tutum/dnsutils`, который доступен в Docker Hub и содержит исполняемые файлы `nslookup` и `dig`.
```shell
kubectl run dnsutils --image=tutum/dnsutils --generator=run-pod/v1 \
--command --sleep infinity
```
```shell
kubectl exec dnsutils nslookup kubia-headless
```
output:
```
Name: kubia-headless.default.svc.cluster.local
Address: 10.108.1.4
Name: kubia-headless.default.svc.cluster.local
Address: 10.108.2.5
```
DNS-сервер возвращает два разных IP-адреса для FQDN 
`kubia-headless.default.svc.cluster.local`. Это IP-адреса двух модулей, которые сообщают о готовности.
___
# 2. Обнаружение даже неготовых модулей
Можно использовать механизм поиска в DNS, который находит даже неготовые модули.
Для того чтобы сообщить Kubernetes, что в службу должны быть добавлены все модули, независимо от состояния готовности модуля, необходимо добавить в службу следующую аннотацию:
```yaml
kind: Service
metadata:
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
```
> [!important] 
> API служб Kubernetes уже поддерживает новое поле секции spec под названием `publishNotReadyAddresses`, которое заменит аннотацию `tolerate-unready-endpoints`. 

___
# Литература
- Марко Лукша – Kubernetes в действии
