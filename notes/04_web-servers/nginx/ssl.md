- **Зачем:** Шифрование трафика между клиентом и сервером [[http-https|(HTTPS)]].

- **Let's Encrypt:** Автоматические бесплатные сертификаты.

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d example.com -d www.example.com
```

- **Конфиг с SSL:**

```nginx
server {
	listen 443 ssl http2;
	server_name example.com;
	ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
	root /var/www/html;
}
```

- **Редирект с HTTP на HTTPS:**

```nginx
server {
	listen 80;
	server_name example.com;
	return 301 https://$host$request_uri;
}
```

#### TLS Терминация

- Nginx использует TLS/HTTPS
- Сервер "за ним" использует HTTP (без SSL)
- Nginx **термирует** TLS шифрование
Расшифровывает сообщение от клиента и на сервер отправляет его расшифрованным.

#### Сквозная передача TLS

- TLS Pass Through
- Сервер "за nginx" работает по TLS
- Nginx проксирует пакеты данных напрямую на сервер (вместе с TLS handshake)
- Nginx имеет доступ только к данным 4 уровня модели [[network-reference-models#Модель OSI (Open Systems Interconnection) — это концептуальная 7-уровневая модель, описывающая стандарты взаимодействия сетевых устройств (от физического кабеля до приложений)|OSI]]


Пример полного конфига:

```bash
upstream go_backend {
        server 127.0.0.1:8080; # Сюда Nginx будет пересылать очищенный трафик
		keepalive 32; # Держать 32 постоянных HTTP/2 соединения с бэкендом
}

server {
        server_name app.kubernetes.local 192.168.100.111;
        listen 80;
        listen [::]:80;
        
# Если клиент зашел по обычному HTTP — принудительно перенаправляем на HTTPS
        return 301 https://$host$request_uri;
}


server {
        server_name app.kubernetes.local;
        listen 443 ssl;
        listen [::]:443 ssl;

# Подключение сертификатов, приватный/публичный ключи
        ssl_certificate /etc/nginx/ssl/application.pem;
        ssl_certificate_key /etc/nginx/ssl/application.key;
        
# Продакшн-настройки шифрования (Защита от старых уязвимых протоколов)
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m; # Кэшировать TLS сессии в памяти
        ssl_session_timeout 1d;

        gzip on;
        gzip_types application/json text/plain text/css;
        gzip_min_length 256;
        
# Ограничиваем максимальный размер загружаемого файла (Защита от DDoS)
        client_max_body_size 10m;
# Буфер для чтения заголовков клиента
        client_header_buffer_size 1k;

        location / {
                proxy_pass http://go_backend; # Шлем пакеты в наш upstream

                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
# Очищаем заголовок, чтобы работал keepalive c Go
                proxy_set_header Connection ""; 
                
                
# Передаем оригинальный домен сайта
                proxy_set_header Host $host;
# Передаем реальный IP клиента
                proxy_set_header X-Real-IP $remote_addr;
# Цепочка прокси-адресов
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
# Передаем бэкенду, что клиент зашел именно по https
                proxy_set_header X-Forwarded-Proto $scheme;

                proxy_connect_timeout 5s;
                proxy_read_timeout 60s;
                proxy_send_timeout 60s;
        }
}
```
