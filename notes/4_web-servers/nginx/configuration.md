#### **Главные конфиги:**
- `/etc/nginx/nginx.conf`
- `/etc/nginx/conf.d/default.conf`

##### Директории, в которых по дефолту хранится статические файлы:
- `usr/share/nginx/html`
- `var/www/html`
- `etc/nginx/sites-aviables`, `etc/nginx/sites-enabled`

##### Директории с логами:
- error_log  `/var/log/nginx/error.log`
- access_log  `/var/log/nginx/access.log`

### **Основные директивы:**

**Структура:** Иерархическая — директивы могут быть на разных уровнях (http, server, location).

- `worker_processes auto;` — сколько процессов использовать (обычно = кол-ву ядер CPU).
- `worker_rlimit_nofile 100000;` — лимит числа файловых дескрипторов
- `events { worker_connections 1024; }` — сколько соединений может держать один воркер.
- `include /etc/nginx/conf.d/*.conf;` — подключать все конфиги из папки.
- `server_name example.com;` — имя виртуального хоста.
- `root /var/www/html;` — корневая папка с файлами.
- `index index.html index.htm;` — какие файлы искать по умолчанию.


**Лучшая практика:** Не пиши всё в `nginx.conf`. Держи каждый сайт в отдельном файле в `/etc/nginx/sites-available/`, а потом создавай симлинк в `sites-enabled/`.