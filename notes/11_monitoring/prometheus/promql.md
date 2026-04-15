**PromQL (Prometheus Query Language)** — это функциональный язык запросов, разработанный специально для работы с данными временных рядов. Он позволяет в реальном времени выбирать, фильтровать и агрегировать метрики. В отличие от SQL, который работает с таблицами, PromQL работает с **векторами** данных.

Prometheus хранит данные как **временные ряды (time series)** — последовательность значений во времени с метками.

```text
http_requests_total{method="GET", status="200", endpoint="/api"} 
  1621248000 → 42
  1621248005 → 45
  1621248010 → 47
```

**Компоненты метрики:**

- **Имя метрики**: `http_requests_total`    
- **Лейблы (labels)**: `method="GET"`, `status="200"` — ключ-значение для фильтрации
- **Timestamp**: время в Unix (секунды)
- **Value**: числовое значение

---

1. Типы данных в PromQL

Результат любого запроса в PromQL относится к одному из четырех типов:

- **Instant vector (Мгновенный вектор):** набор временных рядов, содержащих ровно одно (самое свежее) значение для каждой комбинации меток.
    - _Пример:_ `http_requests_total` (текущее количество запросов).
- **Range vector (Вектор диапазона):** набор временных рядов, содержащий массив значений за определенный период времени. Обозначается длительностью в квадратных скобках.
    - _Пример:_ `http_requests_total[5m]` (все значения за последние 5 минут).
- **Scalar (Скаляр):** простое числовое значение с плавающей точкой (например, `10`).
- **String (Строка):** простая строка (используется редко).

---

2. Фильтрация (Селекторы)

Выборка данных происходит по имени метрики и меткам (labels):

- `process_cpu_seconds_total{job="api-server"}` — точное совпадение.
- `http_requests_total{status!~"2.."}` — исключение по регулярному выражению (все статусы, не начинающиеся на 2).
- Операторы фильтрации: `=` (равно), `!=` (не равно), `=~` (regex совпадение), `!~` (regex не совпадение).

```promql
# Простое имя метрики
http_requests_total

# Фильтрация по лейблу (оператор =)
http_requests_total{method="GET"}

# Неравенство (!=)
http_requests_total{method!="POST"}

# Регулярное выражение (=~)
http_requests_total{method=~"GET|POST"}

# Исключение по регулярке (!~)
http_requests_total{method!~"PUT|DELETE"}

# Комбинация лейблов
http_requests_total{method="GET", status="200"}
```

---

3. Операторы и функции

Самая мощная часть PromQL — возможность трансформации данных. Агрегация (Aggregation) позволяет объединять данные из множества рядов в один:

- `sum(http_requests_total) by (method)` — сумма всех запросов, сгруппированная по HTTP-методу.
- `avg(node_load1)` — средняя загрузка по всем узлам.
- `max`, `min`, `count`, `quantile` — другие популярные операторы.

Функции для Counter (Счетчиков)

Поскольку счетчики только растут, нам важна скорость их роста, а не абсолютное значение:

- **`rate()`**: Вычисляет среднюю скорость роста в секунду на отрезке времени. _Используется чаще всего._
    - `rate(http_requests_total[5m])`
- **`irate()`**: Вычисляет "мгновенную" скорость по двум последним точкам (лучше для графиков с высокой детализацией).
- **`increase()`**: Показывает, на сколько выросло значение за период.
    - `increase(errors_total[1h])` — сколько ошибок упало за час.

Функции для Gauge (Индикаторов)

- **`predict_linear()`**: Предсказывает, какое значение будет у метрики через N секунд (полезно для мониторинга свободного места на диске).
    - `predict_linear(node_filesystem_free_bytes[1h], 3600) < 0` — диск переполнится через час?
- **`delta()`**: Разница между значением в начале и в конце периода.

---

## Основные выражения сбора метрик

В Prometheus, работающем на `http://localhost:9090`. 

```promql
# Статус ноды (1 — работает, 0 — упала)  
up{job="node-exporter"}




### CPU
# Использование CPU в процентах (всего)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Нагрузка на каждое ядро
100 - (avg by (cpu, instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)




### RAM
# Использование RAM в процентах
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Использование swap
(1 - (node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes)) * 100




### DISK
# Заполненность диска в процентах
100 - ((node_filesystem_avail_bytes{mountpoint="/"} * 100) / node_filesystem_size_bytes{mountpoint="/"})

# Скорость записи на диск
rate(node_disk_written_bytes_total[5m])

# Занятое место на диске (в %) 
100 - (node_filesystem_avail_bytes{mountpoint="/"} * 100 / node_filesystem_size_bytes{mountpoint="/"})




### NETWORK
# Входящий трафик (бит/с)
rate(node_network_receive_bytes_total{device="eth0"}[5m]) * 8

# Исходящий трафик (бит/с)
rate(node_network_transmit_bytes_total{device="eth0"}[5m]) * 8

# Сетевой трафик (входящий, Мбит/с):
rate(node_network_receive_bytes_total[5m]) * 8 / 1024 / 1024




### APP
# RPS (запросов в секунду)
rate(http_requests_total[5m])

# Процент ошибок
(sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))) * 100

# 95-й персентиль латентности
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))




### KUBERNETES
# Поды в статусе CrashLoopBackOff
count(kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"})

# Использование CPU подами в namespace
sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (pod)

# Использование памяти контейнерами
sum(container_memory_working_set_bytes{container!=""}) by (container, pod)

```



---