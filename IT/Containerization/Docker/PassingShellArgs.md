___
___
# Tags
#containerization
___
# Содержание
- [[#1. Инструкции ENTRYPOINT и CMD]]
- [[#2. Разница между формами shell и exec]]
- [[#3. Возможность настройки параметров]]
- [[#4. Возможность настройки параметров окружения]]
	- [[#4.1. Передача нескольких переменных]]
	- [[#4.2. Загрузка из .env файл]]
- [[#Литература]]
___
# 1. Инструкции ENTRYPOINT и CMD
В файле Dockerfile две инструкции определяют две части:
- `ENTRYPOINT` определяет исполняемый файл, вызываемый при запуске контейнера;
- `CMD` задает аргументы, которые передаются в точку входа.
Хотя для указания команды, которую требуется выполнить при запуске образа, можно использовать инструкцию `CMD`, правильный способ – сделать это с помощью инструкции `ENTRYPOINT` и указать `CMD` только в том случае, если требуется определить аргументы по умолчанию. После этого образ можно запустить без указания аргументов:
```shell
docker run <образ>
```
или с дополнительными аргументами, которые переопределяют в файле Dockerfile все то, что задано в инструкции CMD: 
```shell
docker run <образ> <аргументы>
```
___
# 2. Разница между формами shell и exec
Обе инструкции поддерживают две разные формы:
- форма `shell` – например, `ENTRYPOINT node app.js`;
- форма `exec` – например, `ENTRYPOINT ["node", "app.js"]`.
Разница между ними заключается в том, вызывается указанная команда внутри оболочки или нет.

`exec`-форма:
- Запускает команду напрямую, **не через оболочку (shell)**.
- Надежнее в плане сигналов и управления процессами (PID 1).
- **`CMD` или аргументы в `docker run` — ДОБАВЛЯЮТСЯ** к `ENTRYPOINT` как параметры.

`shell`-форма:
- Интерпретируется как `/bin/sh -c "node app.js"`.
- Более привычна для людей, но **хуже** для обработки сигналов (например, `SIGTERM`).
- Аргументы из `CMD` или `docker run` **НЕ ДОБАВЛЯЮТСЯ**, а **заменяют всю команду**.
___
# 3. Возможность настройки параметров
Возьмем за основу такую систему:
Скрипт fortune:
```bash
#!/bin/bash
trap "exit" SIGINT
mkdir /var/htdocs
while :
do
	echo $(date) Writing fortune to /var/htdocs/index.html
	/usr/games/fortune > /var/htdocs/index.html
	sleep 10
done
```
Dockerfile:
```Dockerfile
FROM ubuntu:latest
RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT /bin/fortuneloop.sh
```

Добавим переменную INTERVAL и инициализируем ее значением первого аргумента командной строки:
```bash
#!/bin/bash
trap "exit" SIGINT
INTERVAL=$1
echo Configured to generate new fortune every $INTERVAL seconds
mkdir -p /var/htdocs
while :
do
	echo $(date) Writing fortune to /var/htdocs/index.html
	/usr/games/fortune > /var/htdocs/index.html
	sleep $INTERVAL
done
```
Теперь модифицируем Dockerfile:
```Dockerfile
FROM ubuntu:latest
RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT ["/bin/fortuneloop.sh"] # |<- форма exec
CMD ["10"] # |<- инструкция по умолчанию
```
И вы можете переопределить интервал сна, выбранный по умолчанию, передав его в качестве аргумента:
```shell
docker run -it docker.io/luksa/fortune:args 15
```
___
# 4. Возможность настройки параметров окружения
Изменим [[#3. Возможность настройки параметров|этот файл]] на использование переменных окружения, все что нужно сделать - убрать инициализацию переменной `INTERVAL` в скрипте:
```bash
#!/bin/bash
trap "exit" SIGINT
INTERVAL=$1
echo Configured to generate new fortune every $INTERVAL seconds
mkdir -p /var/htdocs
while :
do
	echo $(date) Writing fortune to /var/htdocs/index.html
	/usr/games/fortune > /var/htdocs/index.html
	sleep $INTERVAL
done
```
в таком случае в Dockerfile можно задать ее как `env`-переменную:
```Dockerfile
FROM ubuntu:latest
RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENV INTERVAL=10 # |<- поставили значение по умолчанию
ENTRYPOINT /bin/fortuneloop.sh
```
Переопределить можно теперь при запуске так:
```shell
docker run -e INTERVAL=15 image_name
```
## 4.1. Передача нескольких переменных
```shell
docker run -e VAR1=foo -e VAR2=bar my-app
```
## 4.2. Загрузка из .env файл
Есть файл `.env`:
```ini
USERNAME=admin
PASSWORD=secret123
```
Запускать так:
```shell
docker run --env-file .env my-app
```
___
# Литература
- Марко Лукша – Kubernetes в действии
