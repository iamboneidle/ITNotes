___
___
# Tags
#kubernetes
___
# Содержание
- [[#1. Зачем нужна защита от сбоев]]
- [[#2. Fault injection]]
- [[#3. Timeout и retry]]
- [[#4. Circuit breaking]]
- [[#5. Outlier detection и probes]]
- [[#6. Сочетание механизмов]]
- [[#7. Как механизмы дополняют друг друга]]
- [[#8. Практический порядок настройки]]
___
# 1. Зачем нужна защита от сбоев

Сетевой сбой одного микросервиса может распространиться по цепочке: запросы копятся, соединения занимают потоки, а затем падают и зависимые сервисы. Istio позволяет проверять и ограничивать такое поведение на уровне Envoy.

___
# 2. Fault injection

Искусственные сбои нужны для проверки resilience до production. В `VirtualService` можно добавить задержку или ответ с ошибкой:

```yaml
fault:
  delay:
    percentage: {value: 100}
    fixedDelay: 5s
```

```yaml
fault:
  abort:
    percentage: {value: 50}
    httpStatus: 503
```

Задержку или abort генерирует Envoy, а не приложение. После проверки fault injection нужно удалить.

___
# 3. Timeout и retry

Timeout обрывает слишком долгий запрос:

```yaml
http:
- timeout: 3s
  route:
  - destination: {host: reviews}
```

Retry повторяет временно неудачный запрос:

```yaml
retries:
  attempts: 3
  perTryTimeout: 2s
  retryOn: 5xx,connect-failure
```

Ретраи настраиваются у вызывающего сервиса — его Envoy делает исходящий запрос. Повторять безопасно в первую очередь идемпотентные операции (`GET`). Повтор `POST` может выполнить действие дважды. Слишком много попыток создаёт retry storm; ограничивайте число попыток и `connectionPool.http.maxRetries`. Общий timeout должен учитывать все попытки.

___
# 4. Circuit breaking

`connectionPool` ограничивает число соединений и ожидающих запросов, чтобы перегруженный сервис не тонул в очереди:

```yaml
trafficPolicy:
  connectionPool:
    tcp:
      maxConnections: 100
    http:
      http1MaxPendingRequests: 10
      maxRequestsPerConnection: 10
```

Когда лимит превышен, Envoy быстро возвращает `503`. Это предпочтительнее бесконечного ожидания.

___
# 5. Outlier detection и probes

`outlierDetection` пассивно следит за реальными ответами реплик и временно исключает endpoint, который выдаёт много ошибок:

```yaml
outlierDetection:
  consecutive5xxErrors: 5
  interval: 10s
  baseEjectionTime: 30s
  maxEjectionPercent: 50
```

Kubernetes readiness/liveness — другой механизм. Kubelet активно проверяет health endpoint: readiness убирает под из Endpoints глобально, liveness перезапускает контейнер. Outlier detection действует локально для Envoy и может поймать под, который readiness проходит, но на боевые запросы отвечает ошибками. Механизмы дополняют друг друга.

___
# 6. Сочетание механизмов

Хорошая базовая комбинация: разумный timeout → ограниченные retry для идемпотентных запросов → connection pool против перегрузки → outlier detection для больных реплик. Проверять конфигурацию стоит fault injection'ом и метриками, а параметры подбирать по реальной нагрузке.

___
# 7. Как механизмы дополняют друг друга

```mermaid
flowchart LR
    R["запрос"] --> T["timeout<br>не ждать бесконечно"]
    T -->|"временный сбой"| RT["ограниченный retry"]
    RT -->|"backend перегружен"| CB["circuit breaking"]
    CB -->|"реплика стабильно ошибается"| OD["outlier detection"]
    OD --> H["трафик на здоровые endpoints"]
    style R fill:#673ab7,color:#fff
    style T fill:#326ce5,color:#fff
    style RT fill:#f4b400,color:#000
    style CB fill:#db4437,color:#fff
    style OD fill:#db4437,color:#fff
    style H fill:#0f9d58,color:#fff
```

Настройки должны быть согласованы с приложением. Если приложение уже делает собственные ретраи, Envoy умножит число попыток. Если timeout mesh меньше обычного времени ответа, корректная долгая операция будет прервана. Для write-операций нужны идемпотентные ключи или явный запрет повторов.

Проверять resilience стоит через fault injection и метрики: отдельно наблюдать число попыток, 5xx, таймауты, исключённые endpoints и заполненность connection pool.

___
# 8. Практический порядок настройки

Сначала задают общий timeout, чтобы зависший downstream не удерживал ресурсы бесконечно. Затем добавляют небольшой retry только для безопасных ошибок и идемпотентных методов. После этого ограничивают connection pool и включают outlier detection для сервисов с несколькими репликами.

```yaml
trafficPolicy:
  connectionPool:
    http:
      http1MaxPendingRequests: 20
      maxRequestsPerConnection: 10
  outlierDetection:
    consecutive5xxErrors: 5
    interval: 10s
    baseEjectionTime: 30s
    maxEjectionPercent: 50
```

Значения не являются универсальными: их проверяют нагрузочным тестом. Слишком маленькие лимиты вызовут лишние 503, слишком большие вернут проблему длинных очередей.

___
# 9. Итоги

Timeout ограничивает время, retry скрывает временные ошибки, circuit breaking ограничивает нагрузку, outlier detection убирает больные endpoints. Ни один механизм не заменяет Kubernetes probes; каждый работает на своём уровне.
