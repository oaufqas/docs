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
