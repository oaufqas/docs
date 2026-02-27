# Запустить все сервисы в фоне
docker-compose up -d

# Запустить с пересборкой образов
docker-compose up -d --build

# Запустить только определенные сервисы
docker-compose up -d app db




# Остановить и удалить контейнеры
docker-compose down

# Остановить и удалить с volumes
docker-compose down -v

# Остановить и удалить с images
docker-compose down --rmi all




# Показать логи всех сервисов
docker-compose logs

# Показать логи в реальном времени
docker-compose logs -f

# Показать логи конкретного сервиса
docker-compose logs app




# Показать статус всех сервисов
docker-compose ps

# Показать запущенные сервисы
docker-compose ps --services




# Выполнить команду в сервисе
docker-compose exec app ls -la

# Получить оболочку в сервисе
docker-compose exec app /bin/bash







Практические сценарии

# 1. Собрать образ
docker build -t my-shop:latest .

# 2. Запустить контейнер
docker run -d \
  --name online-shop \
  -p 3000:3000 \
  -e NODE_ENV=production \
  -e DB_HOST=db \
  my-shop:latest

# 3. Проверить логи
docker logs -f online-shop



# Запустить PostgreSQL
docker run -d \
  --name postgres-db \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=secret \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:15

# Подключиться к базе
docker exec -it postgres-db psql -U postgres


# Запустить весь стек
docker-compose up -d

# Проверить статус
docker-compose ps

# Посмотреть логи
docker-compose logs -f app

# Остановить стек
docker-compose down






Полезные команды для разработки

# Скопировать файл из контейнера на хост
docker cp my-container:/app/logs/app.log ./app.log

# Скопировать файл с хоста в контейнер
docker cp config.json my-container:/app/config.json


# Сохранить образ в файл
docker save -o my-app.tar my-app:latest

# Загрузить образ из файла
docker load -i my-app.tar



# Удалить все остановленные контейнеры
docker container prune

# Удалить все неиспользуемые образы
docker image prune -a

# Удалить все неиспользуемые volumes
docker volume prune

# Удалить ВСЕ неиспользуемые объекты
docker system prune -a




# Запуск всех сервисов
docker-compose up -d

# Остановка всех сервисов
docker-compose down

# Просмотр логов
docker-compose logs
docker-compose logs backend  # логи конкретного сервиса

# Пересборка и запуск
docker-compose up -d --build

# Статус сервисов
docker-compose ps
