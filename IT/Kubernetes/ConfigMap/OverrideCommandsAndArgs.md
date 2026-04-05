___
___
# Tags

___
# Содержание
- [[#1. Переопределение команды и аргументов в Kubernetes]]
- [[#2. Настройка переменных среды для контейнера]]
	- [[#2.1. Указание переменных среды в определении контейнера]]
- [[#3. Ссылка на другие переменные среды в значении переменной]]
- [[#Литература]]
___
# 1. Переопределение команды и аргументов в Kubernetes
В Kubernetes при указании контейнера можно переопределять и инструкцию `ENTRYPOINT`, и инструкцию `CMD`. Для этого задайте свойства `command` и `args` в спецификации контейнера:
```yaml
kind: Pod
spec:
  containers:
    – image: some/image
      command: ["/bin/command"]
      args: ["arg1", "arg2", "arg3"]
```
___
# 2. Настройка переменных среды для контейнера
Возьмем пример [[PassingShellArgs#3. Возможность настройки параметров|отсюда]] и передаем bash-скрипт на использование переменных среды:
все что нужно сделать - это убрать инициализацию переменной:
```bash
#!/bin/bash
trap "exit" SIGINT
echo Configured to generate new fortune every $INTERVAL seconds
mkdir -p /var/htdocs
while :
do
	echo $(date) Writing fortune to /var/htdocs/index.html
	/usr/games/fortune > /var/htdocs/index.html
	sleep $INTERVAL
done
```
## 2.1. Указание переменных среды в определении контейнера
[[PassingShellArgs#4. Возможность настройки параметров окружения|Про env в Dockerfile]]

```yaml
kind: Pod
spec:
  containers:
    – image: luksa/fortune:env
      env:               # |<-
        – name: INTERVAL # | передача переменной окружения
          value: "30"    # |__
      name: html-generator
```

> [!important] 
> Как отмечалось ранее, переменная среды устанавливается внутри определения контейнера, а не на уровне модуля. 

___
# 3. Ссылка на другие переменные среды в значении переменной
В предыдущем примере вы задали фиксированное значение переменной среды, но вы также можете ссылаться на ранее заданные переменные среды или любые другие существующие переменные с помощью синтаксической конструкции $(VAR):
```yaml
env:
  - name: **FIRST_VAR**
    value: "foo"
  - name: SECOND_VAR
	value: "$(FIRST_VAR)bar"
```
___
# Литература
- Марко Лукша – Kubernetes в действии
