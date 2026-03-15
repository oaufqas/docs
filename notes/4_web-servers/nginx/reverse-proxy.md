#### **Reverse Proxy**

#### Reverse proxy - это когда Nginx принимает запросы и перенаправляет их на другой сервер (например, Node.js на `localhost:3000`).

- **Зачем:** Обработка статики nginx'ом, балансировка, SSL termination.

- **Пример:**

```nginx
location / {
	proxy_pass http://localhost:5000;
	proxy_set_header Host $host;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header X-Forwarded-Proto $scheme;
}
```
#### **Важные заголовки:**

- `X-Real-IP` — реальный IP клиента.
- `X-Forwarded-For` — список прокси, через которые прошёл запрос.
- `X-Forwarded-Proto` — оригинальный протокол (http/https).