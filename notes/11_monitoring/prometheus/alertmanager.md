**Alertmanager** — это отдельный компонент в экосистеме Prometheus, который берет на себя всю логику по обработке уведомлений.

Если сам Prometheus только **вычисляет** проблему (например: «загрузка CPU > 90%»), то Alertmanager решает, **кому, куда и когда** об этом сообщить.

Зачем он нужен?

В больших системах тысячи метрик. Если Prometheus просто будет слать письмо на каждый «чих», почта инженера превратится в спам-фильтр. Alertmanager решает три главные задачи:

1. **Группировка (Grouping):** Объединяет похожие алерты в одно сообщение. Если упал целый кластер из 20 микросервисов, вы получите одно уведомление «Упал кластер X», а не 20 отдельных алертов.
2. **Подавление (Inhibition):** Отключает алерты, если уже сработал «родительский» алерт. Например, если упал весь дата-центр, Alertmanager не будет слать уведомления о том, что в этом дата-центре недоступны отдельные сервера.
3. **Тишина (Silences):** Позволяет временно заглушить алерты (например, на время планового техобслуживания).

---

Архитектура и этапы обработки

Процесс выглядит так:

1. **Prometheus** постоянно проверяет правила (Rules) и, если условие верно, отправляет алерт в Alertmanager.
2. **Alertmanager** принимает его и прогоняет через цепочку:
    - **Deduplication:** Склеивает дубликаты.
    - **Grouping:** Ждет некоторое время (`group_wait`), чтобы собрать похожие события в одну пачку.
    - **Routing:** Смотрит на метки (labels) алерта и решает, в какой канал его отправить (в Slack для разработчиков или в PagerDuty для дежурных админов).

---

Основные понятия конфигурации

Конфиг Alertmanager строится на **дереве маршрутов (Routes)**:

- **Receivers:** Куда слать? (Slack, Webhook, Email, Telegram, Opsgenie).
- **Routes:** Правила маршрутизации. Например: «Если в алерте есть метка `severity="critical"`, шли в PagerDuty, иначе — в Slack».
- **Repeat Interval:** Как часто повторять уведомление, если проблема всё еще не решена (например, раз в 4 часа).

---

Пример логики в конфиге:

```yaml
route:
  group_by: ['alertname', 'cluster'] # Группируем по имени и кластеру
  group_wait: 30s                  # Ждем 30с перед отправкой первой пачки
  repeat_interval: 3h              # Повтор через 3 часа
  receiver: 'default-slack'        # Куда слать по умолчанию

  routes:
  - match:
      severity: 'critical'         # Если алерт критический...
    receiver: 'pager-duty-sms'     # ...шлем SMS
```

Типовые алерты:

```yaml
# Высокая нагрузка на CPU
- alert: HighCPUUsage
  expr: (100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 80
  for: 10m
  annotations:
    summary: "High CPU usage on {{ $labels.instance }}"

# Диск почти заполнен
- alert: DiskSpaceLow
  expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 10
  for: 5m
  annotations:
    summary: "Disk space low on {{ $labels.instance }}"

# Сервис недоступен
- alert: ServiceDown
  expr: up{job="myapp"} == 0
  for: 1m
  annotations:
    summary: "Service {{ $labels.job }} is down"

# Много 5xx ошибок
- alert: HighErrorRate
  expr: (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))) * 100 > 5
  for: 5m
  annotations:
    summary: "High error rate on {{ $labels.instance }}"
```