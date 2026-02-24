# Шаблон SSL конфигурации для сайта
# Поместить в /etc/nginx/conf.d/example.com.conf

# Редирект HTTP -> HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    # Для Let's Encrypt проверки
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Остальной трафик редиректим на HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS сервер
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;

    # Пути к сертификатам
    ssl_certificate /etc/nginx/ssl/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/example.com/privkey.pem;

    # Дополнительные SSL параметры
    ssl_trusted_certificate /etc/nginx/ssl/live/example.com/chain.pem;
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # Безопасность
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Корень для статики
    root /var/www/html;
    index index.html index.htm;

    # Лимиты
    client_max_body_size 100M;
    client_body_timeout 60s;
    client_body_buffer_size 128k;

    # Основной location
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API прокси
    location /api/ {
        proxy_pass http://app:5000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;

        # Таймауты
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # Буферизация
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        proxy_busy_buffers_size 8k;

        # Лимиты для API
        limit_req zone=login burst=10 delay=5;
    }

    # Статические файлы (кеширование)
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
        try_files $uri =404;
    }

    # Защита от доступа к скрытым файлам
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # Защита от доступа к .git
    location ~ /\.git {
        deny all;
        return 404;
    }

    # Обработка ошибок
    error_page 404 /404.html;
    location = /404.html {
        internal;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        internal;
    }
}