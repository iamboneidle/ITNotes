___
___
# Tags
#kubernetes 
___
# Содержание
- [[#1. Знакомство с ConfigMap]]
- [[#2. Создание ConfigMap]]
	- [[#2.1. Создание с помощью kubectl create]]
	- [[#2.2. Исследование созданного словаря]]
	- [[#2.3. Создание записи словаря конфигурации из содержимого файла]]
	- [[#2.4. Другие варианты]]
		- [[#2.4.1. Задать ключ словаря вручную]]
		- [[#2.4.2. Передать весь каталог]]
		- [[#2.4.3. Объединить разные варианты]]
- [[#Литература]]
___
# 1. Знакомство с ConfigMap
Kubernetes позволяет выделить параметры конфигурации в отдельный объект, называемый ConfigMap, который представляет собой ассоциативный массив, содержащий пары ключ-значение, где значения варьируются от коротких литералов до полных файлов конфигурации.
Приложению не нужно читать словарь конфигурации напрямую или даже знать, что он существует. Содержимое словаря передается в контейнеры либо как переменные среды, либо как файлы в томе. И поскольку на переменные среды можно ссылаться в аргументах командной строки с помощью синтаксической конструкции `$(ENV_VAR)`, то вы также можете передавать записи словаря конфигурации процессам в качестве аргументов командной строки.
___
# 2. Создание ConfigMap
## 2.1. Создание с помощью kubectl create
Записи словаря ConfigMap можно определить, передав литералы команде `kubectl`:
```shell
kubectl create configmap fortune-config --from-literal=sleep-interval=25
```
Словари конфигурации, как правило, содержат более одной записи. Для того чтобы создать словарь конфигурации с несколькими литеральными записями, нужно добавить несколько аргументов `--from-literal`:
```shell
kubectl create configmap myconfigmap \
--from-literal=foo=bar \
--from-literal=bar=baz \
--from-literal=one=two
```
## 2.2. Исследование созданного словаря
Команда:
```shell
kubectl get configmap fortune-config -o yaml
```
output:
```yaml
apiVersion: v1
data:
  sleep-interval: "25" # |<- единственная запись в словаре
kind: ConfigMap
metadata:
  creationTimestamp: 2016-08-11T20:31:08Z
  name: fortune-config # |<- имя словаря
  namespace: default
  resourceVersion: "910025"
  selfLink: /api/v1/namespaces/default/configmaps/fortune-config
  uid: 88c4167e-6002-11e6-a50d-42010af00237
```
Можно написать такой YAML самостоятельно, оставив только:
```shell
apiVersion: v1
data:
  sleep-interval: "25"
kind: ConfigMap
metadata:
  name: fortune-config
```
## 2.3. Создание записи словаря конфигурации из содержимого файла
Словари ConfigMap также могут хранить сложные конфигурационные данные, такие как полные файлы конфигурации:
```shell
kubectl create configmap my-config --from-file=config-file.conf
```
При выполнении приведенной выше команды `kubectl` ищет файл `config-file.conf` в каталоге, в котором вы запустили `kubectl`. Затем он сохранит содержимое файла с ключом `config-file.conf` в словарь конфигурации (имя файла используется в качестве ключа словаря).
## 2.4. Другие варианты
###### 2.4.1. Задать ключ словаря вручную
```shell
kubectl create configmap my-config --from-file=customkey=config-file.conf
```
###### 2.4.2. Передать весь каталог
```shell
kubectl create configmap my-config --from-file=/path/to/dir
```
###### 2.4.3. Объединить разные варианты
```shell
kubectl create configmap my-config \
--from-file=foo.json \
--from-file=bar=foobar.conf \
--from-file=config-opts/ \
--from-literal=some=thing
```
___
# Литература
- Марко Лукша – Kubernetes в действии
