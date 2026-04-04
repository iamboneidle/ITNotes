___
___
# Tags
#linux
___
# Содержание
- [[#Изменение названия ветки в git]]
- [[#Изменение прав доступа к файлу попадает в diff git'а]]
- [[#Поиск в файловой системе]]
- [[#Передача ssh ключа на удаленный хост]]
- [[#Принудительное закрытие соединения с хостом]]
- [[#Скачивание лога установки Jenkins]]
- [[#Скачивание файла с удаленного хоста]]
- [[#Создание ssh ключа]]
- [[#Удаление всех Docker-образов и Docker-контейнеров]]

+[[UsefulUtils|Полезные утилиты]]
___
# Создание ssh ключа
```shell
$ ssh-keygen -t rsa
```

___
# Скачивание лога установки Jenkins
``` shell
curl -u <login>:<password> \
-o build.log \
"https://jenkins.domain.com/job/SOME_JOB/666/consoleText"
```
___
# Скачивание файла с удаленного хоста
```shell
scp user@remote_host:/path/to/file.txt ./local_file.txt
```
___
# Передача ssh ключа на удаленный хост
```shell
ssh-copy-id -i ~/.ssh/my_custom_key.pub user@remote_host
```
___
# Изменение названия ветки в git
#### Локально
```shell
git branch -m <old_name> <new_name>
```
#### В origin
```shell
git push origin :<old_name> <new_name>
```
#### Если коммитит в старую ветку origin
Проверка апстрима:
```shell
git branch -vv
```
Сброс апстрима:
```shell
git branch --unset-upstream
```
Пуш в корректную ветку:
```shell
git push -u origin <new_name>
```
Не забудь очистить ветку в origin.
___
# Удаление всех Docker-образов и Docker-контейнеров
#### Контейнеры:
```shell
docker rm -vf $(docker ps -a -q)
```
#### Образы:
```shell
docker rmi -f $(docker images -a -q)
```
#### Очитка кэша
```shell
docker builder prune --all
docker system prune -a -f
```
- `docker builder prune --all` — удаляет **только кеш сборки** (Buildx).
- `docker system prune -a -f` — удаляет **всё неиспользуемое** (остановленные контейнеры, образы без тегов, сети и кеш сборки) **автоматически** (`-f`) без запроса подтверждения.
___
# Изменение прав доступа к файлу попадает в diff git'а
### Переопределение значения
#### Локальный
```shell
git config core.fileMode false
```
#### Глобальный
```shell
git config --global core.fileMode false
```
### Проверка текущего состояния:
#### Локальный (выполняется в корне git-репозитория):
```shell
git config --get --local core.fileMode
```
#### Глобальный
```shell
git config core.fileMode
```
#### Расшифровки
***Локальный конфиг переопределяет глобальный.***
- `true` – права доступа имеют значение;
- `false` – права доступа не имеют значения.

### Проверка, из какого конфига берутся права:
```shell
git config --show-origin core.fileMode
```
#### Расшифровки
- `file:.git/config        false` – из корня репозитория;
- `file:/home/<user>/.gitconfig	false` – глобальные настройки.
___
# Поиск  в файловой системе
```shell
find . -type <obj_type> -name <file_name>
```
obj_types:
- **`f`** — обычный файл;
- **`d`** — директория (папка);
- **`l`** — символическая ссылка (symlink);
- **`c`** — символьное устройство (character device);
- **`b`** — блочное устройство (block device);
- **`p`** — именованный канал (FIFO, named pipe);
- **`s`** — сокет (socket).
___
# Принудительное закрытие соединения с хостом
Полезно для дебага обработки исключений в работе с Socket'ами.

Узнать ip-адрес по hostname:
```shell
dig +short site.domain.com
```
Просмотр имеющихся соединений:
```shell
sudo conntrack -L | grep -E "<ip>"
```
Блокировка ч/з `iptables` (не дает автоматически переподключиться после сброса соединения):
```shell
sudo iptables -I FORWARD -d <ip> -p tcp --dport <port> -j REJECT
```
Сброс соединения:
```shell
sudo conntrack -D -d <ip> -p tcp --dport <port>
```
Разблокировка в `iptables`:
```shell
sudo iptables -D FORWARD -d <ip> -p tcp --dport <port> -j REJECT
```
___
