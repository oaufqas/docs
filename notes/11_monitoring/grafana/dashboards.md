#### Дашборд — это набор **панелей (Panels)**. Каждая панель может быть разного типа:

- **Time Series:** классический график с линиями (самый частый выбор для метрик CPU/RAM).
- **Stat:** одна большая цифра (например, общее количество онлайн-пользователей).
- **Gauge:** «спидометр», показывающий текущее значение относительно максимума (удобно для заполнения дисков).
- **Table:** обычная таблица (хорошо для списка последних ошибок или логов).
- **Heatmap:** тепловая карта (идеально для визуализации задержек/latency запросов).

### Переменные (Variables)

В верхней части хорошего дашборда всегда есть выпадающие списки (например, `Server`, `Environment`, `Namespace`).

- Вместо того чтобы делать 100 графиков для 100 серверов, ты пишешь запрос в духе: `node_cpu_seconds_total{instance="$server"}`.
- Когда ты меняешь значение в списке, Grafana подставляет его в переменную `$server`, и все графики мгновенно обновляются.

#### Библиотека дашбордов (Community Dashboards)

Почти никогда не нужно рисовать дашборды с нуля для стандартных вещей.

- На сайте grafana.com лежат тысячи готовых решений.
- **Как использовать:** Копируешь ID (например, `1860`), заходишь в свою Grafana -> `Dashboards` -> `New` -> `Import` -> вставляешь ID.
- Можно импортировать дашборд, нажав "Edit" на любой панели и посмотреть, какую сложную формулу написал автор, чтобы получить этот график.

#### Режимы отображения данных

- **Real-time:** Дашборд сам обновляется каждые 5-15 секунд (настраивается в углу экрана).
- **History:** Ты можешь выделить мышкой любой участок на графике, чтобы «зумиться» в него и посмотреть, что именно происходило в момент аварии два дня назад.

#### Дашборды как код (Dashboard as Code)

В серьезном продакшене дашборды не создают «руками» через интерфейс.

- Дашборд — это по сути огромный **JSON-файл**.
- Их хранят в Git и раскатывают их через Terraform или Ansible. Если Grafana упадет или её случайно удалят, дашборды легко восстановить из кода.

--- 

### Настройка своих дашбордов и установка

Чтобы при старте сразу появились свои дашборды, нужно указать на них путь в конфигурационном файле (`/etc/grafana/provisioning/dashboards/dashboards.yml`)

```yml
apiVersion: 1

providers:
  - name: 'Dashboards'
    orgId: 1
    folder: 'Application'
    type: file
    disableDeletion: false
    editable: true
    options:
      # Путь внутри контейнера Grafana, где лежат ваши файлы .json
      path: /etc/grafana/provisioning/dashboards
```

Файл дашборда (`grafana/provisioning/dashboards/cpp_dashboard.json`)

Обычно этот JSON получают, создав график вручную в Grafana, а затем нажав **Share -> Export -> Save to file**.

Ниже представлен минималистичный рабочий шаблон дашборда. В нем настроены два графика: **RPS** (скорость запросов в секунду на базе `http_requests_total`) и **Температура** (на базе `app_temperature_celsius`).

```json
{
  "annotations": { "list": [] },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": null,
  "links": [],
  "liveNow": false,
  "panels": [
    {
      "title": "Application Requests Per Second (RPS)",
      "type": "timeseries",
      "id": 1,
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "Prometheus" },
          "expr": "rate(http_requests_total[1m])",
          "legendFormat": "Method: {{method}}, Status: {{status}}",
          "refId": "A"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "custom": { "drawStyle": "line", "lineInterpolation": "smooth" },
          "unit": "reqps"
        }
      }
    },
    {
      "title": "Device Temperature",
      "type": "gauge",
      "id": 2,
      "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "Prometheus" },
          "expr": "app_temperature_celsius",
          "refId": "A"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "celsius",
          "min": 0,
          "max": 100,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 60 },
              { "color": "red", "value": 80 }
            ]
          }
        }
      }
    }
  ],
  "refresh": "5s",
  "schemaVersion": 38,
  "style": "dark",
  "tags": [],
  "time": { "from": "now-5m", "to": "now" },
  "timepicker": {},
  "timezone": "",
  "title": "C++ Application Metrics",
  "uid": "cpp_metrics_dashboard",
  "version": 1
}
```