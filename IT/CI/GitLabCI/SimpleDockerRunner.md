___
___
# Tags
#ci
___
# Содержание
- 
___
# 1. Введение
Тут разберем, пожалуй, простейший вариант подготовки GitLab Runner'а — внутри Docker-контейнера.
Runner, тем не менее, в моем случае будет работать на удаленном сервере.
___
# 2. Подготовка runner'а
## 2.1. Volume
В начале следует создать volume для runner'а:
```shell
docker volume create runner_volume
```
## 2.2. Запуск runner'а
```shell
docker run -d --name gitlab-runner -v /var/run/docker.sock:/var/run/docker.sock -v runner_volume:/etc/gitlab-runner gitlab/gitlab-runner:latest
```
На этом с runner'ом все.
___
# 3. Регистрация
![](GitlabRunner.png)
Вот тут следует нажать кнопку, которая позволит зарегистрировать runner'а в репозитории.
Далее будет предложено задать теги, описание, а также таймаут для джоб, запускаемых на раннере.
Теперь зарегистрируем раннер:
```shell
docker exec -it gitlab-runner gitlab-runner register
```
Введем url-адрес и полученный токен в настройках. Выберем базовый image, который будет использоваться для выполнения команд. Готово!

Если у вас приватный registry, то следует изменить config.toml, который обитает в созданном volume `runner_volume`:
```toml
concurrent = 1
check_interval = 0
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "test_runner"
  url = "********"
  id = 103
  token = "********"
  token_obtained_at = 2026-04-27T19:33:17Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
      AssumeRoleMaxConcurrency = 0
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    helper_image = "my.private.registry/gitlab/gitlab-runner-helper:<tag>" #|<- вот тут это надо сделать
    tls_verify = false
    image = "ubuntu:22.04"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    volume_keep = false
    shm_size = 0

```
___
# 4. Тестирование runner'а
В целом, достаточно просто запустить минимальный пайплайн, чтобы проверить, что раннер корректно подключился, нашел все необходимые образы и готов к работе:
```yaml
stages:
  - list

show-files:
  stage: list
  script:
    - ls -la
  tags:
    - test

```
___
