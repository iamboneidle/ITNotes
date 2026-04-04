___
___
# Tags
#kubernetes 
___
# Содержание
- [[#1. Создание сертификата TLS для Ingress]]
- [[#Литература]]
___
# 1. Создание сертификата TLS для Ingress
Когда клиент открывает подключение TLS к контроллеру Ingress, контроллер терминирует соединение TLS. Коммуникация между клиентом и контроллером шифруется, тогда как коммуникация между контроллером и бэкенд-модулем – нет. Работающее в модуле приложение не нуждается в поддержке TLS. Например, если в модуле запущен веб-сервер, то он может принимать только HTTP-трафик и давать контроллеру Ingress заботиться обо всем, что связано с
TLS. Для того чтобы позволить данному контроллеру это делать, необходимо прикрепить к Ingress сертификат и закрытый ключ. Они должны храниться в ресурсе Kubernetes под названием Secret, на который затем ссылается манифест ресурса Ingress.

```shell
openssl genrsa -out tls.key 2048
```
```shell
openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj \ /CN=kubia.example.com
```
Затем надо создать Secret из двух файлов:
```shell
kubectl create secret tls tls-secret --cert=tls.cert --key=tls.key
```
Закрытый ключ и сертификат теперь хранятся в секрете tls-secret. Теперь можно обновить объект Ingress, чтобы он также принимал запросы HTTPS для
`kubia.example.com`. Манифест ресурса Ingress теперь должен выглядеть следующим образом:
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  tls: # |<- вся конфигурация TLS находится под этим аттрибутом
    – hosts:
      – kubia.example.com # |<- подключения TLS будут приниматься для этого хоста
        secretName: tls-secret # |<- ключ и сертификат из ранее созданного секрета
  rules:
    – host: kubia.example.com
      http:
        paths:
          – path: /
            backend:
              serviceName: kubia-nodeport
              servicePort: 80
```
___
# Литература
- Марко Лукша – Kubernetes в действии
