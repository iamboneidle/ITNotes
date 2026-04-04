___
___
# Tags
#monitoring 
___
# Содержание
- [[#Описание]]
- [[#Дашборд периодов недоступности]]
- [[#Правило алерта на падение]]
___
# Описание
Тут представлены наработки по мониторингу standalone Docker контейнеров. Интересные дашборды и правила алертинга.
___
# Дашборд периодов недоступности
![[DockerDashboard.png]]
Интересный дашборд, который знает, в какие периоды времени были недоступны те или иные контейнеры на сервере. Получает инфу от CAdvisor'а, динамически находит через его метрики новые контейнеры, не сбрасывая при этом старые.

```json title=dashboard_config.json fold
{
  "id": 521,
  "type": "state-timeline",
  "title": "Статус контейнеров",
  "gridPos": {
    "x": 0,
    "y": 1,
    "h": 8,
    "w": 24
  },
  "fieldConfig": {
    "defaults": {
      "custom": {
        "lineWidth": 0,
        "fillOpacity": 100,
        "spanNulls": false,
        "insertNulls": false,
        "hideFrom": {
          "tooltip": false,
          "viz": false,
          "legend": false
        },
        "axisPlacement": "auto"
      },
      "color": {
        "mode": "fixed",
        "fixedColor": "green"
      },
      "mappings": [
        {
          "options": {
            "1": {
              "color": "green",
              "index": 0,
              "text": "Up"
            }
          },
          "type": "value"
        },
        {
          "options": {
            "match": "null+nan",
            "result": {
              "color": "dark-red",
              "index": 1,
              "text": "Down"
            }
          },
          "type": "special"
        }
      ],
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {
            "color": "green",
            "value": null
          }
        ]
      }
    },
    "overrides": []
  },
  "pluginVersion": "12.3.0",
  "targets": [
    {
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "editorMode": "code",
      "expr": "max by (name) (container_last_seen{instance=~\"$host_instance\", name=~\"$container_name\", container_label_com_docker_compose_project=~\"$compose_project\"} and ((time() - container_last_seen) <= 5)) * 0 + 1",
      "legendFormat": "__auto",
      "range": true,
      "refId": "A"
    }
  ],
  "datasource": {
    "type": "prometheus",
    "uid": "${DS_PROMETHEUS}"
  },
  "options": {
    "mergeValues": true,
    "showValue": "auto",
    "alignValue": "left",
    "rowHeight": 0.9,
    "legend": {
      "showLegend": false,
      "displayMode": "list",
      "placement": "bottom"
    },
    "tooltip": {
      "mode": "single",
      "sort": "none",
      "hideZeros": false
    },
    "perPage": 10
  }
}
```
___
# Правило алерта на падение
Выражение алерта для Prometheus на падение какого-либо контейнера.
```prometheus title=alert_rule_expr fold
(abs(max by (normalized_name) (
  label_replace(
    max_over_time(container_last_seen{instance=~".*some-instance.*", name!="", name!~".*test.*"}[1w]),
    "normalized_name",
    "$1",
    "name",
    "(blue_|green_)?(deploy-.*)"
  )
) - scalar(max(
  max by (normalized_name) (
    label_replace(
      max_over_time(container_last_seen{instance=~".*some-instance.*", name!="", name!~".*test.*"}[1w]),
      "normalized_name",
      "$1",
      "name",
      "(blue_|green_)?(deploy-.*)"
    )
  )
))) > 10) 
or 
(max by (normalized_name) (
  label_replace(
    max_over_time(container_last_seen{instance=~".*some-instance.*", name!="", name!~".*test.*"}[1w]),
    "normalized_name",
    "$1",
    "name",
    "(blue_|green_)?(deploy-.*)"
  )
) < time() - 60)
```
1. `container_last_seen{instance=~".*srv-mops-auto.*",name!="",name!~".*blue.*"}[1w]`
    - Берём метрику времени последнего seen контейнера
    - Только с инстансов `some_instance`
    - Исключаем пустые имена и контейнеры с `blue` в имени
    - За период 1 неделя
2. `max_over_time(...)[1w]`
    - Для каждого контейнера находим максимальное значение за неделю
    - Это даёт время последнего seen каждого контейнера
3. `max by (name) ( ... )`
    - Группируем результаты по имени контейнера
    - Получаем время последнего seen для каждого уникального контейнера
4. `scalar(max( ... ))`
    - Берём максимальное значение из всех контейнеров
    - Преобразуем в скаляр (просто число)
    - Это самое свежее время среди ВСЕХ контейнеров
5. `abs(время_контейнера - самое_свежее_время) > 10`
    - Для каждого контейнера вычисляем разницу с самым свежим временем
    - Берем абсолютное значение (модуль)
    - Если разница >10 секунд → контейнер рассинхронизирован
6. `or` (логическое ИЛИ)
    - Запрос возвращает контейнеры, которые соответствуют ЛЮБОМУ из двух условий
7. `время_контейнера < time() - 60`
    - Второе условие: проверяем, обновлялся ли контейнер за последние 60 секунд
    - Сравниваем время контейнера с текущим временем (`time()`)
    - Если разница > 60 секунд → контейнер "застрял"
___
