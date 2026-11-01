Если Dockerfile собирает один кирпичик (контейнер), то Compose — это инструкция по сборке всего здания. Вместо того чтобы вводить в терминале пять длинных команд docker run с кучей флагов, вы один раз описываете всё в файле docker-compose.yml и запускаете одной командой.

#### Основные возможности:

1. Управление одной командой: Запуск (up) и остановка (down) всей пачки сервисов сразу.
2. Изоляция проектов: Compose создает для каждого проекта отдельную сеть, чтобы контейнеры разных проектов не мешали друг другу.
3. Сохранение данных: Автоматическое создание сетей и томов (volumes), описанных в конфиге.
4. Переменные окружения: Поддержка .env файлов для хранения секретов и настроек.
5. Внутри сети Compose контейнеры видят друг друга по именам сервисов (DNS).


Структура состоит из нескольких корневых блоков верхнего уровня. Каждый блок отвечает за свой изолированный тип ресурсов (контейнеры, диски, сети, секреты)

```yaml
# Версия (version) в современных спецификациях Compose стала необязательной.
name: my_app_project # Имя проекта (опционально)

services: # 1. Список контейнеров
  web:
    image: nginx:latest

volumes: # 2. Постоянные диски
  db_data:

networks: # 3. Изолированные сети
  app_net:

secrets: # 4. Конфиденциальные файлы
  db_password:
```

### Блок `services` (Управление контейнерами)


Каждая запись внутри `services` — это описание будущего контейнера. Здесь настраивается всё: от сборки до лимитов ресурсов. Самые популярные аргументы:

- **`image`** — указывает готовый Docker-образ Docker Hub или локального реестра.

```yml
image: postgres:15-alpine # Версия опциональна, если не указывать - последняя
```

- **`build`** — используется, если образ нужно собрать из исходного кода прямо сейчас.

```yml
build:
  context: ./app # Путь к папке с Dockerfile
  dockerfile: Custom.Dockerfile # Имя файла (если не стандартный Dockerfile)
  args: # Переменные сборки (доступны ТОЛЬКО при сборке)
    BUILD_VERSION: 1.2.0
```

- **`ports`** — проброс портов из контейнера на хост-машину (`ХОСТ:КОНТЕЙНЕР`).

```yml
ports:
  - "8080:80" # Порт 80 внутри контейнера будет доступен как 8080 снаружи
  - "127.0.0.1:9001:9001" # Привязка порта только к локальному интерфейсу
```

- **`environment`** — переменные окружения, которые будут доступны внутри запущенного контейнера.

```yml
environment:
  DEBUG: "true"
  DB_USER: admin # Можно писать напрямую
  APP_KEY: ${SECRET_KEY} # Можно подтягивать из файла .env на хосте
```

- **`env_file`** — позволяет загрузить сразу много переменных из внешнего файла.

```yml
env_file:
  - .env.production
```

- **`depends_on`** — задает порядок запуска сервисов.

```yaml
depends_on:
  db:
	condition: service_healthy # Ждать, пока у сервиса 'db' успешно пройдет healthcheck
```


- **`restart`** — политика автоматического перезапуска при падении приложения.
	- `no` — не перезапускать (по умолчанию).
	- `always` — перезапускать всегда.
	- `on-failure` — перезапускать, только если контейнер упал с кодом ошибки (не 0).
	- `unless-stopped` — перезапускать всегда, кроме случаев, когда пользователь сам его остановил.
- **`healthcheck`** — регулярная проверка работоспособности приложения внутри контейнера.

```yml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/metrics"]
  interval: 30s # Как часто проверять
  timeout: 10s # Сколько ждать ответа
  retries: 3 # Сколько падений подряд означают поломку
  start_period: 40s # Время на прогрев приложения при старте (ошибки игнорируются)
```

- **`deploy`** — управление ресурсами контейнера (лимиты CPU и RAM).

```yml
deploy:
  resources:
	limits: # Максимально доступно
	  cpus: '0.50' # 50% от одного ядра
	  memory: 512M
	reservations: # Гарантированно выделить при старте
	  memory: 256M
```


### Блок `volumes` (Постоянное хранение данных)

Контейнеры по своей природе эфемерны: если удалить контейнер, все данные внутри него сотрутся. **Volumes** переносят данные на жесткий диск хост-машины.

Вариант А: Именованные volumes (Управляются Docker)

Создаются в корневом блоке `volumes`. Удобны для баз данных, так как Docker сам заботится о правах доступа.

```yml
services:
  db:
    image: mysql
    volumes:
      - mysql_data:/var/lib/mysql # Имя_диска : Путь_в_контейнере

volumes:
  mysql-data: # Объявление именованного хранилища (Docker создаст папку в /var/lib/docker/volumes/)
    driver: local
    driver_opts:
      type: none
      device: /data/mysql
      o: bind
      
      
  redis-data:
    driver: local
  uploads-data:
    driver: local 
```

Вариант Б: Bind Mounts (Монтирование конкретной папки с хоста)

Используется без объявления в корневом блоке `volumes`. Отлично подходит для разработки, чтобы прокидывать код в контейнер «на лету».

```yml
services:
  web:
    image: nginx
    volumes:
      - ./html:/usr/share/nginx/html:ro # ро — Read Only (контейнер не смо
```


### Блок `networks` (Изолированные сети)

По умолчанию Docker Compose объединяет все сервисы в одну общую сеть. Но для безопасности инфраструктуру принято разделять. Например, база данных не должна быть в одной сети с внешним миром.

```yml
services:
  web:
    image: nginx
    networks:
      - frontend

  app:
    image: my_backend
    networks:
      - frontend
      - backend # Бекенд находится в обеих сетях и выступает мостом

  db:
    image: postgres
    networks:
      - backend # База полностью изолирована от сети frontend

networks:
  frontend: # Объявление сети frontend
  backend: # Объявление сети backend
    driver: bridge # Драйвер сети (bridge — стандартная виртуальная сеть на одной машине)
```

_Внутри одной сети контейнеры могут общаться друг с другом по их именам в compose-файле (например, в строке подключения к БД на бекенде можно написать `host=db`)._

Расширенный пример:

```
networks:
  frontend-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
  backend-network:
    driver: bridge
    internal: true  # Изолированная сеть без доступа к интернету
  monitoring-network:
    driver: bridge
```


### Блок `secrets` (Защищенная передача конфиденциальных данных)

Передавать пароли, токены и приватные ключи через `environment` небезопасно (их видно в логах, в команде `docker inspect` и в истории git). Для этого существуют **secrets**.

Compose берет файл с секретом на хосте и безопасно монтирует его внутрь контейнера как файл в оперативную память (в директорию `/run/secrets/`).

```yml
services:
  db:
    image: postgres
    environment:
      # Многие официальные образы поддерживают суффикс _FILE
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password 
    secrets:
      - db_password # Даем контейнеру доступ к этому секрету

secrets:
  db_password:
    file: ./secrets/db_pg_pass.txt # Путь к файлу с паролем на вашей машине
```


### Главные команды Compose

- `docker compose up` - Поднимает все сервисы (добавьте `-d` для фона).
- `docker compose down` - Останавливает и удаляет контейнеры и сети проекта. (`-v` удаляет тома)
- `docker compose stop/start` - Просто останавливает/запускает контейнеры (без удаления).
- `docker compose ps` - Показывает статус контейнеров конкретно этого проекта.
- `docker compose logs -f` - Показывает цветной поток логов от всех сервисов сразу.
- `docker compose build` - Пересобирает образы, если вы изменили Dockerfile.
- `docker compose exec [service] [cmd]` - Зайти в конкретный сервис (например, docker compose exec db bash).
- `docker-compose up -d app db` - Запустить только определенные сервисы
- `docker-compose down --rmi all` - Остановить и удалить с images
- `docker-compose config` - Проверить конфиг
- `docker-compose --profile dev up -d` - Запустить определенный профиль

### Пример

```yaml
services:
  # ----- БАЗА ДАННЫХ MySQL -----
  mysql:
    image: mysql:8.0
    container_name: ${PROJECT_NAME:-app}-mysql
    restart: unless-stopped
    
    # Сети
    networks:
      - backend-network
    
    # Порты (не открывать наружу в production!)
    ports:
      - "${MYSQL_HOST_PORT:-3306}:3306"
    
    # Переменные окружения
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-rootpassword}
      MYSQL_DATABASE: ${DB_NAME:-appdb}
      MYSQL_USER: ${DB_USER:-appuser}
      MYSQL_PASSWORD: ${DB_PASSWORD:-apppassword}
      MYSQL_ROOT_HOST: '%'
      TZ: Europe/Moscow
    
    # Файлы окружения
    env_file:
      - ./env/mysql.env
    
    # Тома (volumes)
    volumes:
      - mysql-data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d
      - ./mysql/conf:/etc/mysql/conf.d
      - ./backups:/backups
    
    # Команда запуска с дополнительными параметрами
    command: 
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --max_connections=500
      - --innodb_buffer_pool_size=1G
    
    # Healthcheck (обязательно для depends_on: condition: service_healthy)
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s
    
    # Лимиты ресурсов
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
    
    # Безопасность
    security_opt:
      - no-new-privileges:true
    
    # Логирование
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```


### Готовые compose-файлы для популярных приложений

#### REDIS

```yml
  redis:
    image: redis:7-alpine
    container_name: ${PROJECT_NAME:-app}-redis
    networks:
      - backend-network
    volumes:
      - redis-data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    restart: unless-stopped
```

#### NGINX

```yml
  nginx:
    image: nginx:alpine
    container_name: ${PROJECT_NAME:-app}-nginx
    
    networks:
      - frontend-network
    
    ports:
      - "${HTTP_PORT:-80}:80"
      - "${HTTPS_PORT:-443}:443"
    
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./client/dist:/usr/share/nginx/html:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
    
    depends_on:
      - api
    
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3
    
    restart: unless-stopped
```

#### ADMINER

```yml
  adminer:
    image: adminer:latest
    container_name: ${PROJECT_NAME:-app}-adminer
    networks:
      - frontend-network
      - backend-network
    ports:
      - "${ADMINER_PORT:-8080}:8080"
    environment:
      ADMINER_DEFAULT_SERVER: mysql
    depends_on:
      - mysql
    profiles:
      - dev
      - tools
```

#### PORTAINER

```yml
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    networks:
      - monitoring-network
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer-data:/data
    command: --admin-password $$2y$$10$$YourHashedPasswordHere
    profiles:
      - tools
      - admin
```

#### PROMETHEUS

```yml
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    networks:
      - monitoring-network
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    ports:
      - "9090:9090"
    profiles:
      - monitoring
```

#### GRAFANA

```yml
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    networks:
      - monitoring-network
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_INSTALL_PLUGINS: grafana-piechart-panel
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
    ports:
      - "3001:3000"
    depends_on:
      - prometheus
    profiles:
      - monitoring
```