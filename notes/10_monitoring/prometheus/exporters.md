## Exporter - это мост между системой и Prometheus

- Это отдельный процесс или агент, который собирает метрики из приложения, системы или сервиса и преобразует их в формат, понятный прометеусу ("/metrics")

### Наиболее популярные Exporters:

 - `node_exporter` - Метрики хоста (CPU, память, диски, сеть)
 - `blackbox_exporter` - Проверка доступности (ping, HTTP, TCP)
 - `postgres_exporter` - Состояние PostgreSQL
 - `jmx_exporter` - Java-приложения
 - `nginx_exporter` - Метрики Nginx
 - `redis_exporter` - Метрики Redis