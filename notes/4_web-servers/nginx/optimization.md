#### **Optimization**

- **Кэширование статики:**

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
	expires 30d;
	add_header Cache-Control "public, immutable";
}
```

- **Gzip сжатие:** Включает сжатие перед отправкой клиенту.

```nginx
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
```

- **Буферизация:** Ускорение отдачи статики (`sendfile on; tcp_nopush on;`).
- **Keep-Alive:** Держать соединение открытым для нескольких запросов (`keepalive_timeout 65;`).
- **Лимиты (Защита от DDoS):**

```nginx
limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
limit_req zone=login burst=5;
```