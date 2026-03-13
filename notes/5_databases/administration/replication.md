**Репликация** — это процесс копирования данных с одного сервера базы данных (**Master** или **Source**) на один или несколько других (**Replica** или **Replica**). Простыми словами: это **создание «живой» копии базы, которая обновляется почти одновременно с оригиналом.**

#### Типы репликации:

###### 1. По способу передачи данных

- **Асинхронная (по умолчанию):** Master записал данные и тут же ответил клиенту «Готово». Реплика получит их чуть позже.
    - _Плюс:_ Быстро. _Минус:_ При аварии можно потерять пару секунд данных, которые еще не долетели до реплики.
- **Синхронная:** Master ждет, пока реплика подтвердит получение данных, и только потом отвечает клиенту.
    - _Плюс:_ Нулевая потеря данных. _Минус:_ Если реплика «тупит» или моргнула сеть, Master тоже встанет колом.

###### 2. По логике работы

- **Физическая (Binary):** Копируются байты файлов данных.
    - _Характерно для:_ Postgres. Очень надежно, реплика — точная копия байт-в-байт.
- **Логическая:** Передаются SQL-команды или изменения строк.
    - _Характерно для:_ MySQL (через **Binlog**). Позволяет реплицировать только одну таблицу или базу, а не весь сервер.

---

Как это устроено под капотом

1. **Master** записывает каждое изменение в специальный лог (**Binlog** в MySQL, **WAL** в Postgres).
2. **Replica** подключается к мастеру, читает этот лог и «проигрывает» его у себя.
3. **Lags (Лаги):** Если реплика не успевает записывать данные так же быстро, как мастер, возникает «лаг репликации». Это главная метрика для DevOps.

Что выбрать для своей библиотеки?

- **MySQL:** Популярна схема **Source-Replica** (бывший Master-Slave). Настраивается через `server-id` и передачу позиции бинлога.
- **Postgres:** Чаще используют **Streaming Replication**. Для управления отказоустойчивостью в продакшене обычно ставят надстройку **Patroni**.

---

## Настройки репликации:

## PostgreSQL (Streaming Replication):
Это физическая репликация, копия байт в байт.

#### **На Master сервере:**
1. В [[./optimization#PostgreSQL|postgresql.conf]] разрешаем подключения и настраиваем WAL:

```conf
listen_addresses = '*'
wal_level = replica
max_wal_senders = 10 
```

2. В `pg_hba.conf` разрешаем репликацию для IP реплики:

```conf
host replication replication_user 192.168.1.50/32 md5
```

3. Создаем пользователя в SQL:

```sql
CREATE ROLE replication_user WITH REPLICATION LOGIN PASSWORD 'password';
```

4. Перезапускаем: `systemctl restart postgresql`.


#### **На Replica сервере:**

1. Останавливаем базу и удаляем старые данные (Data Dir):

```bash
systemctl stop postgresql
rm -rf /var/lib/postgresql/14/main/*
```

2. Клонируем данные с мастера (Base Backup):

```bash
pg_basebackup -h 192.168.1.40 -D /var/lib/postgresql/14/main/ -U replication_user -P -R
```

_Флаг `-R` создаст файл `standby.signal`, который скажет Postgres работать как реплика._
3. Запускаем: `systemctl start postgresql`.

---

## MySQL (Binary Log Replication):
Классическая репликация в MySQL — логическая (через Binlog).

#### **На Master сервере:**

1. В [[./optimization#MySQL|my.cnf]] задаем ID и включаем логи:

```conf
[mysqld]
server-id = 1
log-bin = mysql-bin
```

2. Перезапускаем и создаем пользователя в SQL:

```sql
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';
FLUSH PRIVILEGES;
```

3. Смотрим текущую позицию лога (запомни `File` и `Position`):

```sql
SHOW MASTER STATUS;
```


**На Replica сервере:**

1. В `my.cnf` задаем уникальный ID:

```conf
[mysqld]
server-id = 2
```

2. Перезапускаем и настраиваем связь в SQL:

```sql
CHANGE MASTER TO
  MASTER_HOST='192.168.1.40',
  MASTER_USER='repl_user',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001', -- из шага 3
  MASTER_LOG_POS=154;                 -- из шага 3
```

3. Запускаем:

```sql
START SLAVE;
SHOW SLAVE STATUS\G -- ищи Slave_IO_Running: Yes
```