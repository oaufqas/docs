Запуск баз данных в Docker — это стандарт для разработки и стейджинга, но в продакшене это требует осторожности (нужно следить за производительностью дисков и сетью).

- **Ephemeral vs Persistent**: Контейнеры по своей природе временны, поэтому к каждому контейнеру с базой данных нужно создавать **Volumes**, чтобы не потерять данные.
- **Process ID 1**: Внутри контейнера процесс БД (mysqld или postgres) должен иметь PID 1, чтобы Docker мог корректно передавать сигналы завершения (SIGTERM) для чистого выключения.
- **Network**: Базы нужно выносить в отдельную Docker-сеть (`internal`), чтобы они не светили портами в интернет.

При запуске базы данных в docker container'е, ей нужно указать обязательные переменные, такие как root пароль, иначе будет выдавать ошибку. Также после запуска нужно делать healthcheck, чтобы другие сервисы не падали, например docker-compose.yml:

```yml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres"] # Для Postgres
  # test: ["CMD", "mysqladmin" ,"ping", "-h", "localhost"] # Для MySQL
  interval: 10s
  timeout: 5s
  retries: 5
```


`/docker-entrypoint-initdb.d/` - при запуске можно пробросить папку с `.sql` или `.sh` скриптами через volume, docker выполнит их при первом запуске контейнера.

---

### Run PostgreSQL in docker container

**Обязательные переменные окружения (`-e`):**

- `POSTGRES_PASSWORD`: Пароль для суперпользователя (обязателен).
- `POSTGRES_USER`: Имя суперпользователя (по умолчанию `postgres`).
- `POSTGRES_DB`: Имя базы, которая создастся при старте (по умолчанию совпадает с пользователем).

**Директория для Volume:**

- `/var/lib/postgresql/data` — здесь лежат все таблицы, логи и конфиги.

**Команда запуска:**

```bash
docker run -d \
  --name my-postgres \
  -e POSTGRES_PASSWORD=mysecret \
  -v /my/own/datadir:/var/lib/postgresql/data \
  -v /init.sql:/docker-entrypoint-initdb.d \
  -p 5432:5432 \
  postgres:15
```

---

### Run MySQL in docker container

**Обязательные переменные окружения (`-e`):**

- `MYSQL_ROOT_PASSWORD`: Пароль root-пользователя (обязателен).
- `MYSQL_DATABASE`: Имя создаваемой базы при старте.
- `MYSQL_USER` и `MYSQL_PASSWORD`: Создание обычного пользователя (не root).

**Директория для Volume:**

- `/var/lib/mysql` — основная папка с данными.

**Команда запуска:**

```bash
docker run -d \
  --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=mysecret \
  -e MYSQL_DATABASE=app_db \
  -v /my/mysql/data:/var/lib/mysql \
  -v /init.sql:/docker-entrypoint-initdb.d \
  -p 3306:3306 \
  mysql:8.0
```

---

### Run Redis in docker container (In-memory Key-Value)

Самый простой в запуске. Данные хранит в оперативной памяти, но может сбрасывать на диск (RDB/AOF).

- **Обязательные переменные:** По умолчанию пароля нет. Чтобы задать его, используется флаг команды запуска.
- **Директория Volume:** `/data`

**Команда запуска:**

```bash
docker run -d \
  --name my-redis \
  -p 6379:6379 \
  -v /my/redis/data:/data \
  redis:7-alpine \
  redis-server --requirepass "mypassword" --appendonly yes
```

- **Нюанс:** Флаг `--appendonly yes` включает сохранение данных на диск при каждой записи (защита от потери при рестарте).

---

### Run MongoDB in docker container (NoSQL Document DB)

Хранит данные в формате BSON (похоже на JSON). Очень популярна для логов и гибких структур.

- **Обязательные переменные:**
	- `MONGO_INITDB_ROOT_USERNAME`: Логин админа.
	- `MONGO_INITDB_ROOT_PASSWORD`: Пароль админа.
- **Директория Volume:** `/data/db`

**Команда запуска:**

```bash
docker run -d \
  --name my-mongo \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  -v /my/mongo/data:/data/db \
  -p 27017:27017 \
  mongo:6.0
```

- **Нюанс:** Монго очень чувствительна к файловой системе. На сетевых дисках (NFS) может работать нестабильно.

---

### Run ClickHouse in docker container (Columnar OLAP DB)

Аналитическая база данных. Летает на огромных объемах данных (миллиарды строк), но не любит частые точечные обновления (`UPDATE/DELETE`).

- **Обязательные переменные:**
    - `CLICKHOUSE_USER`: Имя пользователя.
    - `CLICKHOUSE_PASSWORD`: Пароль.
    - `CLICKHOUSE_DB`: Имя БД по умолчанию.
- **Директории Volume:**
    - `/var/lib/clickhouse` — сами данные (**самое важное**).
    - `/var/log/clickhouse-server` — логи.
- **Команда запуска:**

```bash
docker run -d \
  --name my-clickhouse \
  -e CLICKHOUSE_USER=devops \
  -e CLICKHOUSE_PASSWORD=password \
  --ulimit nofile=262144:262144 \
  -p 8123:8123 -p 9000:9000 \
  -v /my/ch/data:/var/lib/clickhouse \
  clickhouse/clickhouse-server:latest
```


- **Нюанс:** ClickHouse требует больших лимитов на открытие файлов. Флаг `--ulimit nofile=262144:262144` обязателен, иначе база упадет под нагрузкой.