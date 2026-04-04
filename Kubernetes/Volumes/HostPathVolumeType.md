___
___
# Tags
#kubernetes 
___
# Содержание
- [[#1. Введение]]
- [[#2. Исследование системных модулей с томами]]
- [[#Литература]]
___
# 1. Введение
Том `hostPath` указывает на определенный файл или каталог в файловой системе узла. Модули, работающие на одном узле и использующие один и тот же путь в томе `hostPath`, видят одни и те же файлы.
Тома `hostPath` – это тип постоянного хранения, потому что содержимое томов `gitRepo` и `emptyDir` удаляется, когда модуль демонтируется, в то время как содержимое тома `hostPath` остается. Если модуль удаляется и следующий модуль использует том hostPath, который указывает на тот же самый путь на хосте, новый модуль увидит все, что было оставлено предыдущим модулем, но только если он назначен на тот же узел, что и первый модуль.
Лучше не использовать для хранения файлов БД, так как она может гулять по кластеру с узла на узел.
___
# 2. Исследование системных модулей с томами
Рассмотрим системные модули:
```shell
kubectl get pod s --namespace kube-system
```
output:
```
READY                       STATUS      RESTARTS AGE
fluentd-kubia-4ebc2f1e-9a3e 1/1 Running 1        4d
fluentd-kubia-4ebc2f1e-e2vz 1/1 Running 1        31d
```
```shell
kubectl describe po fluentd-kubia-4ebc2f1e-9a3e --namespace kube-system
```
output:
```
Name: fluentd-cloud-logging-gke-kubia-default-pool-4ebc2f1e-9a3e
Namespace: kube-system
...
Volumes:
  varlog:
    Type: HostPath (bare host directory volume)
	Path: /var/log
  varlibdockercontainers:
	Type: HostPath (bare host directory volume)
	Path: /var/lib/docker/containers
```
___
# Литература
- Марко Лукша – Kubernetes в действии
