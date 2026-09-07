___
___
# Tags
#kubernetes
___
# Содержание
- [[#1. Gateway как точка входа]]
- [[#2. VirtualService и правила маршрутизации]]
- [[#3. DestinationRule и группы версий]]
- [[#4. Как ресурсы работают вместе]]
- [[#5. Внутренний трафик и типичные ошибки]]
- [[#6. Проверка связки ресурсов]]
- [[#7. Детали Gateway и VirtualService]]
___
# 1. Gateway как точка входа

`Gateway` описывает, какие порты, протоколы и hostnames слушает ingress gateway. Сам по себе он не выбирает backend и не маршрутизирует запрос к приложению.

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: main-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts: ["myapp.local"]
```

`selector` обязан совпадать с метками подов нужного gateway. Для HTTPS добавляются `tls` и сертификат, часто через Secret.

___
# 2. VirtualService и правила маршрутизации

`VirtualService` описывает, куда отправлять уже принятый запрос: по host, URI, методу, заголовкам и другим условиям. Правила HTTP проверяются сверху вниз, срабатывает первое подходящее.

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews-vs
spec:
  hosts: [reviews]
  http:
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        host: reviews
        port:
          number: 8080
```

Без `match` задаётся маршрут по умолчанию. В `route` можно указать несколько destination с весами, timeout, retries, mirror и заголовками.

___
# 3. DestinationRule и группы версий

`DestinationRule` задаёт политику после выбора сервиса: балансировку, connection pool, TLS и subsets. Subset — это подмножество подов Kubernetes Service, отобранное по меткам.

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews-dr
spec:
  host: reviews
  subsets:
  - name: v1
    labels: {version: v1}
  - name: v2
    labels: {version: v2}
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
```

`host` должен совпадать, а имя subset — с ссылкой из `VirtualService`. Если под не попадает под selector Kubernetes Service, он не станет endpoint subset.

___
# 4. Как ресурсы работают вместе

Путь внешнего запроса выглядит так: Gateway принимает host и порт → VirtualService выбирает destination и subset → DestinationRule описывает subset и политику → Envoy отправляет запрос на IP подходящего пода.

Kubernetes Service при этом не исчезает: он даёт DNS-имя и исходный набор endpoints. Istio накладывает поверх него правила маршрутизации и политики.

___
# 5. Внутренний трафик и типичные ошибки

Для pod-to-pod-трафика используется зарезервированный gateway `mesh`. Если `gateways` не указан, VirtualService по умолчанию действует для mesh-трафика. Чтобы одно правило работало и снаружи, и изнутри, указывают оба gateway и оба host.

Частые ошибки:

- selector Gateway не совпадает с метками ingress gateway;
- в VirtualService указан subset, которого нет в DestinationRule;
- используется короткое имя сервиса из другого namespace вместо FQDN;
- VirtualService создан, но host запроса не входит в `hosts`;
- правило для внешнего трафика забыли привязать к gateway.

___
# 6. Проверка связки ресурсов

```mermaid
flowchart LR
    C["клиент"] --> G["Gateway<br>host и port"]
    G --> V["VirtualService<br>маршрут"]
    V --> D["DestinationRule<br>subset и policy"]
    D --> P["IP подов"]
    style C fill:#673ab7,color:#fff
    style G fill:#f4b400,color:#000
    style V fill:#326ce5,color:#fff
    style D fill:#673ab7,color:#fff
    style P fill:#0f9d58,color:#fff
```

Для сервиса в другом namespace лучше использовать полное имя, например `reviews.production.svc.cluster.local`. Короткое имя разрешается относительно namespace, в котором создан ресурс Istio, и может указывать не на тот Service.

```bash
kubectl get gateway,virtualservice,destinationrule
kubectl get svc,endpointslice reviews
istioctl analyze -n production
```

`istioctl analyze` помогает найти отсутствующие ссылки и конфликтующие host, но не заменяет проверку реального HTTP-запроса и конфигурации Envoy.

___
# 7. Детали Gateway и VirtualService

Gateway отвечает только за входной listener. Он не создаёт Kubernetes Service и не выбирает subset. Связь с маршрутизацией появляется через поле `gateways` в VirtualService.

```yaml
spec:
  hosts: ["myapp.local"]
  gateways: [main-gateway]
  http:
  - match:
    - uri:
        prefix: /reviews
    route:
    - destination:
        host: reviews
        subset: v1
```

Порядок правил важен: специфичные условия ставят выше общего маршрута без `match`. Иначе общий маршрут перехватит запрос раньше и до правила для пути или заголовка дело не дойдёт.

___
# 8. Итоги

Gateway отвечает за вход, VirtualService — за выбор маршрута, DestinationRule — за группы версий и политику назначения. Kubernetes Service остаётся источником DNS и endpoints.
