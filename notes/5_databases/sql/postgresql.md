**PostgreSQL** (Postgres) — это стандарт «золотого сечения» благодаря предсказуемости, расширяемости и строгому следованию стандартам SQL. 

Версии 16 или 17 на **Ubuntu** имеет свои нюансы: лучше использовать официальный репозиторий PGDG, чтобы иметь доступ к свежим версиям и расширениям.

### 1) Подготовка и установка напрямую на сервер

Стандартные репозитории Ubuntu часто содержат устаревшие версии, добавление официального репозитория :

```bash
# Импортируем ключ репозитория
sudo apt install curl ca-certificates -y
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org

# Добавляем репозиторий в список источников
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] http://apt.postgresql.org $(lsb_release -cs)-pgdg main" > /etc/mysql/sources.list.d/pgdg.list'

# Обновляем и устанавливаем (например, 16-ю версию)
sudo apt update
sudo apt install postgresql-16 -y
```

### 2) Управление службой

```bash
sudo systemctl enable postgresql
sudo systemctl status postgresql
```

### 3) Настройка доступа (Самый важный этап)

В Postgres доступ управляется файлом `pg_hba.conf` (Host-Based Authentication).

1. **Разрешить внешние подключения** в основном конфиге:  
    `sudo nano /etc/postgresql/16/main/postgresql.conf`  
    Найдите: `#listen_addresses = 'localhost'` → Измените на: `listen_addresses = '*'`
2. **Настроить права доступа** в `pg_hba.conf`:  
    `sudo nano /etc/postgresql/16/main/pg_hba.conf`  
    Добавьте строку в конец для доступа по паролю из вашей сети:  
    `host all all 10.0.0.0/24 scram-sha-256`  
    _(Где 10.0.0.0/24 — ваша внутренняя сеть)._
3. **Перезапуск**: `sudo systemctl restart postgresql`

### 4) Создание пользователя и БД

В Postgres по умолчанию есть системный пользователь `postgres`. Работаем через него:

```sql
# Заходим под системным пользователем
sudo -u postgres psql

# В консоли psql:
CREATE DATABASE my_project_db;
CREATE USER devops_admin WITH ENCRYPTED PASSWORD 'secure_pass';
GRANT ALL PRIVILEGES ON DATABASE my_project_db TO devops_admin;

-- В Postgres 15+ нужно дополнительно дать права на схему public:
\c my_project_db
GRANT ALL ON SCHEMA public TO devops_admin;
\q
```

### 5) Полезные пути

- **Конфиги**: `/etc/postgresql/16/main/` (тут и `postgresql.conf`, и `pg_hba.conf`).
- **Данные**: `/var/lib/postgresql/16/main/` (тут лежат сами таблицы и WAL-логи).
- **Логи**: `/var/log/postgresql/postgresql-16-main.log`.

### 6) Быстрая проверка производительности

Инструмент `pg_test_fsync` покажет, насколько быстро ваша дисковая подсистема пишет логи транзакций (критично для облачных дисков).

---


#### Архитектура: Процессная модель

В отличие от MySQL (потоки/threads), Postgres на каждое соединение создает отдельный **процесс (OS process)**.

- **Проблема**: Большое количество соединений быстро «съедает» RAM и нагружает CPU на переключение контекста.
- **Решение**: Всегда используйте **Connection Pooler** (стандарт — **PgBouncer**, реже Odyssey). Без пулера база «ляжет» при резком росте трафика.

#### Главные параметры тюнинга ([[../administration/optimization#PostgreSQL|postgresql.conf]])

- **`shared_buffers`**: Память для кэширования данных. Ставить **25%** от общего объема RAM (Postgres полагается и на кэш ОС).
- **`work_mem`**: Память на один запрос (сортировки, join). Считается как `(Total RAM / max_connections)`. Если мало — запросы пишут на диск (медленно).
- **`max_connections`**: Не ставьте больше 200–500 без пулера.
- **`wal_level`**: Для репликации и бэкапов должен быть `replica` или `logical`.

#### MVCC и Vacuum (Боль серминов)

Postgres не удаляет данные физически сразу (`UPDATE` создает новую версию строки, старая помечается «мертвой»).

- **Bloat**: Раздувание таблиц из-за накопления мертвых строк.
- **Autovacuum**: Фоновый процесс очистки. **DevOps-задача**: мониторить, чтобы он успевал чистить быстрее, чем приложение пишет данные. Если база растет, а данных мало — тюньте `autovacuum_vacuum_scale_factor`.

#### [[../administration/replication#PostgreSQL (Streaming Replication)|Репликация]] и HA

- **Streaming Replication**: Основной метод (физическая репликация). Передает WAL-логи (бинарные изменения файлов).
- **Logical Replication**: Позволяет реплицировать отдельные таблицы или базы между разными версиями Postgres.
- **HA Stack**: Для автоматического failover (переключения при сбое) стандартом является связка **Patroni** + **etcd** + **HAProxy**.

#### [[../administration/backup-restore#Команды для бэкапа PostgreSQL|Бэкапы]]: Больше чем дамп

- **`pg_dump`**: Логический бэкап. Хорош для мелких БД.
- **WAL-G / Barman / pgBackRest**: Инструменты для **архивирования WAL** и создания инкрементальных физических бэкапов. Позволяют сделать **PITR** (Point-in-Time Recovery) — восстановление на любую секунду в прошлом.

#### Расширения (Extensions)

Postgres — это конструктор. Самые важные:

- **PostGIS**: Работа с гео-данными.
- **TimescaleDB**: Тайм-серии (метрики).
- **pg_stat_statements**: **Обязательно** для включения. Позволяет увидеть самые тяжелые и медленные запросы в базе.

#### Мониторинг (Grafana + Prometheus)

Используйте `postgres_exporter`. Ключевые метрики:

1. **Transaction rate** (TPS).
2. **Deadlocks** (взаимные блокировки).
3. **Replica Lag** (отставание реплики в байтах/секундах).
4. **Database Size** и темп роста Bloat.