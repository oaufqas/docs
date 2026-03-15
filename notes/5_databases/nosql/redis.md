#### **Redis**

- **Что это:** In-memory хранилище **ключ-значение**. Сверхбыстрое.
    
- **Зачем:** Кэш, сессии, очереди, лимиты.
    
- **Структуры данных:** Строки, списки (list), хеши (hash), сеты (set), сортированные сеты (sorted set).
    
- **Команды:** `SET`, `GET`, `EXPIRE` (таймаут ключа), `INCR` (атомарный инкремент — для счётчиков).
    
- **Персистентность (сохранение на диск):**
    
    - **RDB (снэпшоты):** Бэкап раз в N минут.
        
    - **AOF (append-only file):** Логирует каждую операцию.
        
- **Sentinel и Cluster:** Режимы отказоустойчивости и масштабирования.

---

### **Установка Redis на сервер (Ubuntu/Debian)**

Самый простой и рекомендуемый способ — через официальный APT-репозиторий:

**Шаг 1: Добавление официального репозитория Redis**

```bash
sudo apt-get install lsb-release curl gpg

curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

sudo chmod 644 /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
```

**Шаг 2: Установка Redis**

```bash
sudo apt-get update
sudo apt-get install redis-server
```

**Шаг 3: Проверка и управление сервисом**

```bash
# Проверить статус
sudo systemctl status redis-server
# Включить автозапуск (обычно включено по умолчанию)
sudo systemctl enable redis-server
# Перезапустить после изменений конфига
sudo systemctl restart redis-server

```

### **Настройка Redis (конфигурация)**

Файл конфигурации: `/etc/redis/redis.conf`
**Основные параметры:**

```bash
# 1. Разрешить внешние подключения (осторожно, с паролем!)
bind 0.0.0.0
# 2. Установить пароль
requirepass yourStrongPassword123
# 3. Ограничение памяти (обязательно для production!)
maxmemory 2gb
maxmemory-policy allkeys-lru  # стратегия удаления старых ключей
# 4. Включение AOF для надежности
appendonly yes
# 5. Включение systemd-управления
supervised systemd
# 6. Логи
logfile /var/log/redis/redis-server.log
```
### **Проверка работы Redis**

```bash
# Подключиться к локальному Redis
redis-cli

# Если установлен пароль
redis-cli -a yourStrongPassword123

# Внутри CLI
> ping
PONG
> set test hello
OK
> get test
"hello"

# Проверить удаленное подключение
redis-cli -h <IP-сервера> -p 6379 -a <пароль>
```
### **Безопасность Redis в production**

1. **Никогда не открывайте Redis напрямую в интернет** без пароля и фаервола.

2. **Настройте UFW**:

```bash
sudo ufw allow from <доверенный-IP> to any port 6379
sudo ufw enable
```

3. **Используйте нестандартный порт** (измените `port` в конфиге).

4. **Для облачных серверов** — настройте security groups.

5. **Рассмотрите использование Redis Sentinel или Redis Cluster** для отказоустойчивости