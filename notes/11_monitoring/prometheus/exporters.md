## Exporter - это мост между системой и Prometheus

- Это отдельный процесс или агент, который собирает метрики из приложения, системы или сервиса и преобразует их в формат, понятный прометеусу ("/metrics")

### Наиболее популярные Exporters:

 - `node_exporter` - Метрики хоста (CPU, память, диски, сеть) -- `1860`
 - `blackbox_exporter` - Проверка доступности (ping, HTTP, TCP) -- `7587`
 - `postgres/mysql_exporter` - Состояние PostgreSQL/MySQL -- (PG)`9628` (My) `7362`
 - `jmx_exporter` - Java-приложения
 - `nginx VTS Exporter` - Метрики Nginx -- `2949`
 - `redis_exporter` - Метрики Redis -- `763`
 - `kube_state_metrics` - Состояние Kubernetes -- `13332`
 - `cAdvisor` - Контейнеры docker -- `14282`
 - `Windows Exporter` - Железо Windows (аналог node_exporter) -- `14694`

Если нужно мониторить сервис, у которого нет экспортера, можно использовать библиотеку SDK внутри кода (python, go, java), чтобы сервис сам отдавал метрики на эндпоинт `/metrics`, если изменять код нельзя, то можно написать свой скрипт-экспортер на Python.

#### Использование exporters в yaml конфиге:

```yaml
scrape_configs:
  - job_name: 'linux-servers'
    static_configs:
      - targets: ['192.168.1.10:9100', '192.168.1.11:9100']

  - job_name: 'containers-metrics'
    static_configs:
      - targets: ['localhost:8080'] # Порт cAdvisor по умолчанию
```