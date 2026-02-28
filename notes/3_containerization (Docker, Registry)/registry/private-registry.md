# Чтобы работать с реестром, нужно понимать формат тегов: registry_url/repository_name:tag.

Команда	        Описание
docker login - Авторизация в реестре (нужна для приватных репозиториев или пуша).
docker tag [image] [user]/[repo]:[tag] - Подготовка образа (переименование) для отправки в конкретный репозиторий.
docker push [user]/[repo]:[tag] - Загрузка вашего образа в облако.
docker logout - Выход из учетной записи реестра.



### Self-hosted (Свой собственный реестр)

Если вы не хотите хранить образы в облаке, вы можете запустить собственный реестр прямо в Docker.
Существуют различные решения с открытым исходным кодом для организации container registry. Наиболее популярное из них называется **registry**. Оно **также упаковано в контейнер** и позволяет хранить и распространять образы контейнеров и артефакты.

# Чтобы запустить registry на сервере:
1. Установите Docker и Docker Compose на сервер.
2. Настройте и запустите контейнер с registry.
3. Запустите nginx для обработки TLS и перенаправления запросов в контейнер с registry.
4. Установите TLS-сертификаты и настройте домен.


### Установка:

docker-compose.yaml:
```yaml
services:
  registry:
    image: registry:latest
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/registry.password
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
    volumes:
      # Монтируем файл паролей.
      - ./registry/registry.password:/auth/registry.password
      # Монтируем директорию с данными.
      - ./registry/data:/data
    ports:
      - 5000
  nginx:
    image: nginx:latest
    depends_on:
      - registry
    volumes:
      # Монтируем конфигурацию nginx.
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      # Монтируем сертификаты, полученные от Let's Encrypt.
      - ./nginx/certs:/etc/nginx/certs
    ports:
      - "443:443"
```

nginx.conf:
```conf
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    upstream registry {
        server registry:5000;
    }

    server {
        server_name registry.pliutau.com;
        listen 443 ssl;

        ssl_certificate /etc/nginx/certs/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/privkey.pem;

        location / {
            # Важный параметр для больших образов.
            client_max_body_size                1000m;

            proxy_pass                          http://registry;
            proxy_set_header  Host              $http_host;
            proxy_set_header  X-Real-IP         $remote_addr;
            proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header  X-Forwarded-Proto $scheme;
            proxy_read_timeout                  900;
        }
    }
}
```

mkdir -p ./registry/data

# Устанавливаем htpasswd.
sudo apt install apache2-utils

# Создаём файл password. Имя пользователя: busy, пароль: bee.
htpasswd -Bbn busy bee > ./registry/registry.password

docker-compose up

Теперь вы сможете пушить образы на свой сервер