#### **Главные конфиги:**

- `/etc/nginx/nginx.conf`
- `/etc/nginx/conf.d/default.conf`
- `etc/nginx/sites-aviables`, `etc/nginx/sites-enabled`

**Лучшая практика:** не писать всё в `nginx.conf`, а держать каждый сайт в отдельном файле в `/etc/nginx/sites-available/`, а потом [[inodes-links|создавать симлинк]] в `sites-enabled/`.

Директории, в которых по дефолту хранится статические файлы:

- `usr/share/nginx/html`
- `var/www/html`

Директории с логами:

- error_log  `/var/log/nginx/error.log`
- access_log  `/var/log/nginx/access.log`

#### В каких основных режимах может работать Nginx?

Nginx — это хамелеон, его поведение полностью меняется в зависимости от директив в блоке `location`:

Режим А. Статический Веб-сервер (Static Web Server)

- **Суть:** Nginx сам лезет на SSD хоста, забирает файлы (`.html`, `.css`, `.jpg`, видео) и отдает их клиенту. Делает это быстрее, чем любая другая программа в мире, благодаря флагу `sendfile on` (технология Zero-Copy ядра Linux, когда файл перекачивается из контроллера диска в сетевую карту напрямую силами ядра, вообще не нагружая оперативную память процесса Nginx).

Режим Б. [[reverse-proxy|Обратный Прокси (Reverse Proxy)]]

- **Суть:** Nginx сам не хранит файлы, он работает «щитом» и диспетчером перед вашим реальным бэкендом (например, приложением на Node.js, Python/FastAPI или Go).
- **Настройка:** Реализуется через директиву `proxy_pass`:


```nginx
location /api/ {
	proxy_pass http://127.0.0.1:8000; # Перенаправить запрос на локальное приложение Python
	proxy_set_header Host $host;      # Передать бэкенду реальное имя сайта
	proxy_set_header X-Real-IP $remote_addr; # Передать бэкенду реальный IP-адрес
}
```


Режим В. [[load-balancing|Балансировщик нагрузки (Load Balancer L7)]]

- **Суть:** Распределяет HTTP-запросы между группой (пулом) серверов.
- **Настройка:** Использует контекст `upstream`:

```nginx
upstream backend_pool {
	server 192.168.100.11:8080 weight=3; # Этот сервер мощнее, шлем ему в 3 раза больше трафика
	server 192.168.100.12:8080;
}
server {
	listen 80;
	location / {
		proxy_pass http://backend_pool; # Шлем трафик на пул серверов
	}
}
```

Режим Г. FastCGI / UWSGI прокси

- **Суть:** Общение с бэкендами не по протоколу HTTP, а по специальным бинарным протоколам (актуально для PHP-FPM или Python uWSGI). Использует директивы `fastcgi_pass`.

---

### **Основные директивы:**

Конфигурация Nginx строится по древовидному принципу секций (контекстов). Главный файл лежит по пути `/etc/nginx/nginx.conf`. Иерархическая структура — директивы могут быть на разных уровнях (http, server, location).

Карта наследования контекстов:

```text
  [ Main Context ]  <── Глобальные настройки (Процессы, логи ядра)
         │
         ▼
  [ Events Context ] <── Сетевые лимиты процессов
         │
         ▼
  [ HTTP Context ]   <── Всё, что связано с веб-логикой
         │
         ├─> [ Upstream Context ] <── Пулы бэкендов для балансировки
         │
         └─> [ Server Context ]   <── Виртуальный хост (Ваш сайт)
                   │
                   └─> [ Location Context ] <── Конкретный URL-путь
```

Правила синтаксиса:

1. Каждая простая директива **обязана заканчиваться точкой с запятой `;`**. Забытая `;` — это 99% причин падения теста конфигурации (`nginx -t`).
2. Блочные директивы зажимаются в фигурные скобки `{ ... }` и создают новый **контекст**.
3. Переменные в Nginx всегда начинаются со знака доллара (например, `$remote_addr`).
4. Символ решетки `#` означает комментарий.


```nginx
# Глобальный контекст (Main)
user www-data; # От какого пользователя запускать worker-processes
worker_processes auto; # Автоматически подстроится под ядра CPU вашего моноблока
pid /run/nginx.pid;

# Контекст событий (Связь с ядром Linux)
events {
    worker_connections 1024; # Сколько ОДНОВРЕМЕННЫХ клиентов может держать ОДИН Worker
    use epoll;               # Включаем самый быстрый асинхронный метод ядра Linux
}

# Сетевой HTTP контекст (Веб-логика)
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    log_format main '$remote_addr - $remote_user [$time_local] "$request"';
    access_log /var/log/nginx/access.log main;

    # Включаем оптимизацию ядра Linux для прямой перекачки файлов с SSD в сеть, минуя RAM
    sendfile on; 
    tcp_nopush on;

    # Контекст Виртуального Хоста (Ваш конкретный сайт)
    server {
        listen 80;                         # Слушать порт 80 (HTTP)
        server_name app.kubernetes.local;  # На какой домен откликаться

        # 5. Контекст Локации (Что делать с конкретным URL путем)
        location / {
            root /var/www/html;          # Где физически лежат файлы сайта на SSD
            index index.html index.htm;
        }

        location /images/ {
            root /data/media;    # Картинки забираем из другой папки
            expires 30d;         # Приказываем браузеру закэшировать их на 30 дней
        }
    }
}
```

>**Глобальный контекст (Main)**
- `user www-data;` — определяет, от имени какого непривилегированного пользователя операционной системы Linux будут запускаться рабочие процессы (воркеры). Запуск воркеров от `root` строго запрещен по безопасности.
- `worker_processes auto;` — указывает количество рабочих процессов. Режим `auto` заставляет Nginx прочитать файл `/proc/cpuinfo` вашего хоста и создать ровно столько воркеров, сколько физических ядер имеет процессор (включая потоки гипервизора в `Ring -1`).
- `error_log /var/log/nginx/error.log warn;` — путь к файлу системных ошибок и уровень их детализации (`debug`, `info`, `notice`, `warn`, `error`, `crit`, `alert`, `emerg`).

>**Контекст событий (`events { ... }`)**
- `worker_rlimit_nofile 100000;` — лимит числа файловых дескрипторов
- `events { worker_connections 1024; }` — сколько соединений может держать один воркер.Суммарная емкость сервера рассчитывается как `worker_processes` * `worker_connections`.

>**Контекст HTTP (`http { ... }`)**
- `include /etc/nginx/mime.types;` — подключает внешнюю таблицу соответствия расширений файлов и их типов данных (чтобы браузер понимал, что файл `.css` — это стили, а не просто текст).
- `sendfile on;` — активирует технологию Zero-Copy для раздачи файлов, которую мы разобрали выше.
- `tcp_nopush on;` — заставляет Nginx отправлять HTTP-заголовки ответа и начало файла **одним единым сетевым пакетом TCP**, а не дробить их. Это экономит сетевой трафик и ускоряет загрузку сайтов.
- `keepalive_timeout 65;` — сколько секунд сервер будет держать сетевой сокет открытым после того, как клиент скачал страницу, в ожидании новых запросов. Защищает от необходимости тратить ресурсы процессора на постоянное переоткрытие TLS-хэндшейков.
- `gzip on;` — включает динамическое сжатие текстовых ответов (HTML, CSS, JS) алгоритмом Deflate перед отправкой клиенту по сети. [[optimization_variables|Уменьшает размер файлов в 3-5 раз]], экономя Wi-Fi трафик, но немного нагружает CPU мастера.

>**Контекст Виртуального Хоста (`server { ... }`)**
- `server_name example.com;` — имя виртуального хоста.
- `listen 80;` / `listen 443 ssl;` — какой порт на сетевой карте должен занять этот виртуальный хост.
- `server_name app.kubernetes.local;` — ключевая директива. Nginx читает заголовок `Host` внутри входящего HTTP-пакета. Если там написано `app.kubernetes.local`, запрос полетит именно в этот блок `server`.

>**Контекст Локации (`location { ... }`)**
- `root /var/www/html;` — абсолютный путь в файловой системе Linux, где лежат файлы сайта. Если клиент запрашивает `/index.html`, Nginx полезет искать файл по пути `/var/www/html/index.html`.
- `index index.html index.htm;` — какой файл отдавать по умолчанию, если клиент обратился к корню сайта (просто ввел `/`)
- `try_files $uri $uri/ =404;` — директива проверки существования файлов. Сначала ищет точный файл по пути `$uri`, если его нет — ищет папку `$uri/`, а если и её нет — перенаправляет клиента на стандартную ошибку `404 Not Found`.


**Валидировать написанные конфиги можно так:**

```bash
nginx -t
```

Когда клиент присылает запрос (например, `GET /images/logo.png`), Nginx должен выбрать, какой именно блок `location` внутри вашего `server` запустить. Это одна из самых запутанных тем.

Nginx использует строгие **префиксы** для сопоставления URL, которые имеют жесткий приоритет выполнения (от высшего к низшему):

|Префикс|Тип совпадения|Пример|Описание|Приоритет|
|---|---|---|---|---|
|**`=`**|Точное совпадение|`location = /`|Срабатывает только если URL строго равен `/`. Никакие `/index.html` сюда не попадут.|**1 (Высший)**|
|**`^~`**|Лучший префикс|`location ^~ /images/`|Если URL начинается с этой строки, Nginx **прекращает** дальнейший поиск и берет эту локацию. Регулярные выражения игнорируются.|**2**|
|**`~`**|Регулярное выражение (С учетом регистра)|`location ~ \.(jpg\|png)$`|Ищет совпадение по регулярке. Знак `$` означает конец строки. Поймает все файлы картинок.|**3**|
|**`~*`**|Регулярное выражение (БЕЗ учета регистра)|`location ~* \.pdf$`|Поймает и `.pdf`, и `.PDF`.|**3**|
|_(пусто)_|Обычный префикс|`location /docs/`|Срабатывает, если URL просто начинается с этой строки. Самый ленивый поиск.|

---

#### Эталонный Nginx-конфиг с объяснениями

```nginx
user www-data;                  # Имя непривилегированного пользователя в Linux
worker_processes auto;          # 1 Worker на 1 физическое ядро CPU (максимум скорости кремния)
pid /run/nginx.pid; # Куда master-process запишет свой собственный PID
worker_rlimit_nofile 65535;     # Повышаем лимит дескрипторов файлов ОС для процессов Nginx

events {
    worker_connections 4096;    # Сколько сокетов может держать ОДИН Worker
    use epoll;                  # Асинхронный событийный движок ядра Linux (неблокирующий ввод-вывод)
    multi_accept on;            # Worker будет принимать все новые соединения сразу, а не по одному
}

http {
    # Базовые утилиты и типы данных
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Магия оптимизации ввода-вывода (I/O)
    sendfile on;     # Технология Zero-Copy (копирование файлов с диска минуя RAM)
    tcp_nopush on;    # Отправка HTTP-заголовков и начала файла одним TCP-пакетом
    tcp_nodelay on;             # Отключение алгоритма Нагла (запрет буферизации мелких пакетов, снижает пинг)

    # Настройки таймаутов ленивых соединений
    keepalive_timeout 65;       # Сколько секунд держать открытым TLS-сокет после скачивания страницы
    keepalive_requests 100;     # Макс. количество запросов в рамках одного keep-alive соединения
    reset_timedout_connection on; # Закрывать сокеты для клиентов, которые перестали отвечать

    # Форматирование и оптимизация логов
    log_format production '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for" '
                          'rt=$request_time uct="$upstream_connect_time" uht="$upstream_header_time"';

    access_log /var/log/nginx/access.log production;
    error_log /var/log/nginx/error.log warn;

    # Безопасность: Скрываем версию Nginx от сканеров уязвимостей
    server_tokens off;

    # Динамическое сжатие трафика (Gzip)
    gzip on;
    gzip_comp_level 5;          # Баланс между качеством сжатия текстов и нагрузкой на CPU
    gzip_min_length 256;        # Не сжимать файлы меньше 256 байт
    gzip_proxied any;           # Сжимать запросы даже для проксированных клиентов
    gzip_types text/plain text/css application/json application/javascript text/xml;

    # Зоны кэширования и лимитов (Защита от DDoS)
    # Ограничиваем количество запросов: 20 запросов в секунду с одного IP
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=20r/s;
    
    # Кэширование ответов от бэкендов в оперативной памяти/диске хоста
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m use_temp_path=off;


    upstream go_cluster {
# Алгоритм балансировки: IP Hash (клиент с одним IP всегда летит на один бэкенд)
        ip_hash; 
        
        server 127.0.0.1:8080 max_fails=3 fail_timeout=10s;
        server 127.0.0.1:8081 max_fails=3 fail_timeout=10s;
        keepalive 32;      # Постоянный пул открытых HTTP-соединений с бэкендом
    }

    # Редирект с HTTP на HTTPS
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name _;
        return 301 https://$host$request_uri;
    }

    # Главный защищенный сервер
    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name app.kubernetes.local;

        # TLS/SSL Сертификаты (Паспорт сайта)
        ssl_certificate /etc/nginx/ssl/app.crt;
        ssl_certificate_key /etc/nginx/ssl/app.key;

        # Современные стандарты криптографии (Защита от взлома TLS)
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 1d;

        # Защитные заголовки браузера (Security Headers)
# Защита от кликджекинга
        add_header X-Frame-Options "DENY" always;        
# Запрет браузеру угадывать MIME-тип
        add_header X-Content-Type-Options "nosniff" always;
# Включение XSS-фильтра браузера
        add_header X-XSS-Protection "1; mode=block" always;
# Принудительный HTTPS в браузере (HSTS)
        add_header Strict-Transport-Security "max-age=31536000" always; 

        # Ограничения на загрузку данных
# Максимальный размер загружаемого файла (например, аватарки)
        client_max_body_size 20m;   
        
        # Пример 1: Раздача статики (Фронтенд, картинки)
        location /static/ {
            root /var/www/app;
            expires 30d;            # Кеширование в браузере клиента на месяц
            add_header Cache-Control "public, no-transform";
            access_log off;         # Не забивать логи статического мусора
        }

        # Пример 2: Умный Reverse-Proxy к бэкенду
        location /api/ {
            # Применяем лимит запросов против DDoS (всплеск до 5 запросов разрешен)
            limit_req zone=api_limit burst=5 nodelay;

            proxy_pass http://go_cluster; # Пересылаем очищенный трафик на пул Go

            # Настройка заголовков для бэкенда (Проброс реальности)
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Включаем серверное кэширование ответов от бэкенда
            proxy_cache my_cache;
            proxy_cache_valid 200 10m;  # Успешные ответы кэшировать на 10 минут
            proxy_cache_valid 404 1m;   # Ошибки 404 кэшировать на 1 минуту
            proxy_cache_use_stale error timeout http_500 http_502; # Отдать старый кэш, если бэкенд умер!
            
            # Настройки таймаутов общения с Go
            proxy_connect_timeout 5s;
            proxy_read_timeout 30s;
            proxy_send_timeout 30s;
        }

        # Кастомная страница ошибок
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root /usr/share/nginx/html;
        }
    }
}
```

- **Событийный слой (`events`)**: Ядро через `epoll` просыпается, Worker забирает пакет, проверяет, не превышен ли лимит `worker_connections`.
- **Маршрутизация по домену (`server_name`)**: Nginx считывает HTTP/HTTPS заголовок. Если имя домена совпадает с `app.kubernetes.local`, он запускает этот блок `server`. Если клиент пришел по HTTP, блок `listen 80` выдает мгновенный ответ `301 Moved Permanently`, принудительно переключая браузер на шифрованный порт `443`.
- **Безопасность (`ssl`)**: Внутри `listen 443 ssl` ядро Nginx само берет на себя тяжелую математику расшифровки трафика с помощью ключей. Бэкенд (Go-приложение) об этом даже не думает.
- **Сортировка путей (`location`)**:
    - Если вы запрашиваете картинку `/static/logo.png`, Nginx включает режим **Static Web Server**, через `sendfile` копирует файл с SSD прямо в сетевой сокет, ставит флаг кэша браузера `expires 30d` и выключает логи, экономя ресурс диска хоста.
    - Если вы бьете в `/api/v1/stats`, Nginx проверяет ваш IP по таблице `limit_req_zone`. Если вы шлете 100 запросов в секунду, он выдаст вам ошибку `503 Service Unavailable`. Если лимит чист, Nginx превращается в **Reverse Proxy**, склеивает метаданные (дописывает реальный IP в заголовки) и по стабильному HTTP-каналу шлет запрос в ваш `upstream go_cluster`.