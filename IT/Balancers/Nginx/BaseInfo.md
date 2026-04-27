___
___
# Tags
#balancers 
___
# Содержание
- [[#1. Архитектура Nginx]]
- [[#2. Основные команды]]
- [[#3. Простой конфиг]]
- [[#4. Настройка Reverse Proxy]]
___
# 1. Архитектура Nginx
![[NginxArch]]

___
# 2. Основные команды
#### Версия nginx
```shell
nginx -v
```
#### Проверка синтаксиса
```shell
nginx -t
```
#### Релоад без остановки
```shell
nginx -s reload
```
#### Мягка остановка
```shell
nginx -s stop
```
___
# 3. Простой конфиг
```nginx
user www-data;                # пользователь, под которым работает nginx
worker_processes auto;        # число воркеров (auto - по числу ядер)

events {
	worker_connections 1024;  # макс. число соединений на одного воркера
}

http {
	include mime.types;       # подключение типов файлов (css, html, js...)

	server {                  # блок настроек конкретного сайта
		listen 80;            # порт (обычно 80 или 443)
		server_name site.com; # доменное имя сайта

		location / {          # правило обработки запросов к корню сайта
			root /var/www;    # папка, в которой лежат файлы сайта
			index index.html; # файл, который отдаем по умолчанию
		}
	}
}
```
___
# 4. Настройка Reverse Proxy
```nginx
# блок upstream определяет группу бэкендов
upstream backend_api {
	server 10.0.0.1; # первый сервер
	server 10.0.0.2; # второй сервер
}

server {
	listen 80;
	server_name site.com;
	
	location / {
		# пересылаем запрос на группу upstream
		proxy_pass http://backend_api;
		# важно: передаем реальные заголовки
		proxy_set_header Host $host;
	}
}
```
1. Создаем upstream;
2. Прописываем его в prox y_pass;
3. Nginx сам будет распределять запросы по очереди (Round Robin).

Как это работает:
- Сначала собираем серверы в одну группу (пул) и даем имя backend_api;
- Когда приходит запрос, Nginx не обрабатывает его сам, а пересылает в эту группу http://backend_api;
- Вместе с запросом мы передаем реальный домен (Host), чтобы бэкенд понимал, какой сайт открывать.
___


