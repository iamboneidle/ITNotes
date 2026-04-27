___
___
# Tags
#ci
___
# Содержание
- [[#1. Общая архитектура]]
- [[#2. Типы Runner'ов]]
	- [[#2.1. Shell runner (на хосте)]]
	- [[#2.2. Docker runner (executor = docker)]]
	- [[#2.3. Kubernetes runner]]
- [[#3. Runner как Docker контейнер]]
	- [[#3.1. Что происходит с docker.sock]]
- [[#4. Жизненный цикл одного Job'а (Docker executor)]]
- [[#5. Helper Image — что это и зачем]]
- [[#6. image в pipeline — приоритеты]]
- [[#7. Services — что это]]
- [[#8. Docker-in-Docker (DinD)]]
- [[#9. Volumes и кэш]]
- [[#10. Полная схема — всё вместе]]
___
# 1. Общая архитектура

```
┌─────────────────────────────────────────────┐
│                 GitLab Server               │
│  - хранит .gitlab-ci.yml                    │
│  - создаёт pipeline при push/MR/schedule    │
│  - раздаёт jobs runner'ам через API         │
│  - принимает логи и артефакты обратно       │
└─────────────────┬───────────────────────────┘
                  │  HTTPS (polling)
       ┌──────────▼──────────┐
       │    GitLab Runner     │  ← отдельный процесс/контейнер
       │  (агент на сервере)  │
       └──────────┬──────────┘
                  │
       ┌──────────▼──────────┐
       │  Executor           │  ← способ запуска job
       │  (docker / shell /  │
       │   k8s / ssh / ...)  │
       └─────────────────────┘
```

**Ключевые моменты:**
- GitLab Server никогда сам не запускает код — он только раздаёт задачи
- Runner **сам опрашивает** (polling) GitLab каждые несколько секунд: "есть работа?"
- Связь всегда инициируется runner'ом → GitLab не нужен доступ к runner'у, только наоборот
___
# 2. Типы Runner'ов

| Тип | Где регистрируется | Кому доступен |
|---|---|---|
| **Shared** | На уровне GitLab instance | Всем проектам |
| **Group** | На уровне группы | Всем проектам группы |
| **Project** | На уровне проекта | Только одному проекту |

## 2.1. Shell runner (на хосте)
```
[Сервер]
  └── процесс gitlab-runner
          └── команды script выполняются прямо на хосте
```
- Всё выполняется в окружении хоста
- Нет изоляции между job'ами
- Быстро, просто, но грязно — зависимости накапливаются на хосте
- `image:` в pipeline **игнорируется**
## 2.2. Docker runner (executor = docker)
```
[Сервер]
  └── процесс/контейнер gitlab-runner
          └── для каждого job создаёт новый Docker контейнер
                  └── script выполняется внутри контейнера
```
- Чистая изоляция — каждый job в своём контейнере
- `image:` указывает какой контейнер создать
- Требует Docker daemon
## 2.3. Kubernetes runner
```
[K8s кластер]
  └── Pod: gitlab-runner
          └── для каждого job создаёт новый Pod
                  └── script выполняется внутри Pod'а
```
- Автомасштабирование из коробки
- `image:` указывает образ для Pod'а
___
# 3. Runner как Docker контейнер

```bash
docker run -d \
  --name gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \  # ← КЛЮЧЕВОЙ момент
  -v /etc/gitlab-runner:/etc/gitlab-runner \
  gitlab/gitlab-runner:latest
```
## 3.1. Что происходит с docker.sock

```
[Хост]
  ├── Docker daemon  ←──────────────────────┐
  ├── /var/run/docker.sock                  │ пробрасывается
  └── контейнер: gitlab-runner              │
          └── видит /var/run/docker.sock ───┘
                  └── может отдавать команды Docker daemon'у хоста
```
**Результат:** runner, находясь внутри контейнера, создаёт "соседние" контейнеры на хосте (не дочерние). Все job-контейнеры — братья runner'а, а не его дети.
```
[Хост Docker]
  ├── gitlab-runner (контейнер)
  ├── job-ubuntu-22.04 (контейнер)   ← создан runner'ом для job
  └── job-helper (контейнер)         ← служебный, создан runner'ом
```
___
# 4. Жизненный цикл одного Job'а (Docker executor)
```
1. GitLab создаёт job, runner забирает его по polling

2. Runner читает config.toml → определяет executor = docker

3. Runner запрашивает у Docker: поднять helper image
   └── registry.gitlab.com/gitlab-org/gitlab-runner/gitlab-runner-helper:...

4. Runner запрашивает у Docker: поднять image из pipeline (или дефолт из config.toml)
   └── например ubuntu:22.04

5. Helper контейнер:
   ├── клонирует репозиторий
   ├── скачивает артефакты предыдущих stages
   └── монтирует всё в основной контейнер

6. Основной контейнер (ubuntu:22.04):
   └── выполняет команды из script: построчно

7. После завершения:
   ├── helper собирает артефакты и кэш
   ├── отправляет логи в GitLab
   └── оба контейнера удаляются
```
___
# 5. Helper Image — что это и зачем
Helper — служебный контейнер, **обязательный** для Docker executor. Запускается параллельно с основным контейнером.
**Что делает helper**:
- Клонирует репозиторий (git clone);
- Загружает/выгружает кэш;
- Загружает/выгружает артефакты;
- Управляет передачей файлов между job'ами.

**Почему он отдельный контейнер**:
В основном контейнере (ubuntu:22.04) может не быть git, curl и других инструментов. Helper — минимальный статически слинкованный бинарник от GitLab, который умеет делать всё служебное.
___
# 6. image в pipeline — приоритеты

```
job-level image        ← высший приоритет
      ↓
global image в pipeline
      ↓
image в config.toml    ← fallback / дефолт
```

```yaml
image: node:20           # глобальный — для всех jobs

build:
  script: npm run build  # использует node:20

test:
  image: python:3.12     # переопределяет → python:3.12
  script: pytest

deploy:
  image: bitnami/kubectl  # переопределяет → kubectl
  script: kubectl apply -f k8s/
```
---
# 7. Services — что это

`services` — дополнительные контейнеры, которые запускаются рядом с основным на время job'а. Они доступны по имени сервиса как hostname.
```yaml
job:
  image: python:3.12
  services:
    - postgres:15       # доступен как hostname "postgres"
    - redis:7           # доступен как hostname "redis"
  script:
    - python manage.py test
```
```
[Сеть job'а]
  ├── основной контейнер python:3.12
  ├── postgres:15  ← hostname: postgres
  └── redis:7      ← hostname: redis
```
Все контейнеры в одной сети, видят друг друга по имени сервиса.
___
## 8. Docker-in-Docker (DinD)
Когда в job'е нужно **самому запускать Docker команды** — например, собирать образ:
```yaml
job_build:
  image: docker:24
  services:
    - docker:dind          # отдельный Docker daemon
  variables:
    DOCKER_HOST: tcp://docker:2375  # куда обращаться
  script:
    - docker build -t myapp .
    - docker push my.registry.com/myapp
```
**Как это устроено**
```
[Основной контейнер: docker:24]
  └── docker build .  ← Docker CLI (клиент)
          │
          │ TCP :2375
          ▼
[Service контейнер: docker:dind]
  └── dockerd  ← Docker daemon
          └── здесь реально собирается образ
```
`DOCKER_HOST=tcp://docker:2375` — говорит Docker CLI не искать локальный сокет, а подключиться к daemon'у в service-контейнере по сети.

**DinD vs docker.sock**

| | DinD | docker.sock |
|---|---|---|
| Изоляция | Полная — свой daemon | Нет — доступ к хостовому daemon |
| Безопасность | Требует `privileged: true` | Даёт root на хосте по сути |
| Производительность | Медленнее | Быстрее |
| Типичное применение | Shared runners | Self-hosted runners |
**Почему DinD требует privileged**
DinD запускает полноценный Docker daemon внутри контейнера. Для этого нужны расширенные права ядра (namespaces, cgroups). Без `privileged: true` daemon не стартует.
```toml
# config.toml
[runners.docker]
  privileged = true  # нужно для DinD
```
___
## 9. Volumes и кэш
```toml
[runners.docker]
  volumes = ["/cache"]     # монтируется в каждый job-контейнер
```
```yaml
# pipeline
job:
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - node_modules/    # сохраняется между запусками
```
```
Job 1: node_modules/ → сохраняется в /cache на хосте
Job 2: /cache → node_modules/ восстанавливается (cache hit)
```
___
## 10. Полная схема — всё вместе

```
GitLab Server
  │
  │ push → создаёт pipeline
  │
  ▼
Pipeline (.gitlab-ci.yml)
  ├── stage: build
  │     └── job: build_app
  │           image: node:20
  │           script: npm run build
  │
  ├── stage: test
  │     └── job: run_tests
  │           image: python:3.12
  │           services: [postgres:15]
  │           script: pytest
  │
  └── stage: deploy
        └── job: push_image
              image: docker:24
              services: [docker:dind]
              script: docker build && docker push

Runner (твой контейнер с docker.sock)
  │
  ├── job build_app:
  │     ├── поднимает helper-контейнер
  │     ├── поднимает node:20
  │     ├── helper клонирует репо
  │     ├── node:20 выполняет npm run build
  │     └── контейнеры удаляются
  │
  ├── job run_tests:
  │     ├── поднимает helper-контейнер
  │     ├── поднимает python:3.12
  │     ├── поднимает postgres:15 (service)
  │     ├── helper клонирует репо + артефакты
  │     ├── python:3.12 выполняет pytest
  │     └── контейнеры удаляются
  │
  └── job push_image:
        ├── поднимает helper-контейнер
        ├── поднимает docker:24
        ├── поднимает docker:dind (service)
        ├── docker:24 → docker build → daemon в dind
        └── контейнеры удаляются
```
___
