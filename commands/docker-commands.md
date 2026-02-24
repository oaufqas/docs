# Собрать образ с тегом
docker build -t my-app:latest .

# Собрать с другим Dockerfile
docker build -f Dockerfile.prod -t my-app:prod .

# Собрать без кэша
docker build --no-cache -t my-app:latest .




# Показать все образы
docker images

# Показать образы с фильтром
docker images | grep node

# Показать только ID образов
docker images -q




# Скачать официальный образ
docker pull nginx:alpine

# Скачать из Docker Hub
docker pull username/my-app:latest


# Загрузить в Docker Hub
docker push username/my-app:latest




# Удалить образ по ID/тегу
docker rmi my-app:latest

# Удалить все неиспользуемые образы
docker image prune -a




# Запустить контейнер в фоне
docker run -d --name my-container nginx:alpine

# Запустить с пробросом портов
docker run -d -p 80:80 --name web nginx:alpine

# Запустить с переменными окружения
docker run -d -e NODE_ENV=production --name app my-app:latest

# Запустить с монтированием volumes
docker run -d -v /host/path:/container/path --name app my-app:latest

# Запустить и получить интерактивную оболочку
docker run -it ubuntu:20.04 /bin/bash




# Показать работающие контейнеры
docker ps

# Показать ВСЕ контейнеры (включая остановленные)
docker ps -a

# Показать только ID контейнеров
docker ps -q




# Запустить остановленный контейнер
docker start my-container

# Остановить контейнер
docker stop my-container

# Перезапустить контейнер
docker restart my-container

# Принудительно остановить
docker kill my-container




# Выполнить команду в работающем контейнере
docker exec my-container ls -la

# Получить интерактивную оболочку
docker exec -it my-container /bin/bash

# Выполнить команду от имени пользователя
docker exec -u root -it my-container /bin/bash




# Показать логи контейнера
docker logs my-container

# Показать логи в реальном времени
docker logs -f my-container

# Показать последние N строк
docker logs --tail 100 my-container

# Показать логи с временными метками
docker logs -t my-container




# Удалить остановленный контейнер
docker rm my-container

# Удалить контейнер с принудительной остановкой
docker rm -f my-container

# Удалить все остановленные контейнеры
docker container prune




# Показать статистику всех контейнеров
docker stats

# Показать статистику конкретных контейнеров
docker stats container1 container2




# Показать процессы контейнера
docker top my-container




# Показать всю информацию о контейнере
docker inspect my-container

# Показать только IP адрес
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-container

# Показать информацию об образе
docker inspect my-app:latest




# Показать все сети
docker network ls

# Создать сеть
docker network create my-network

# Подключить контейнер к сети
docker network connect my-network my-container

# Отключить контейнер от сети
docker network disconnect my-network my-container

# Показать детали сети
docker network inspect my-network

# Удалить сеть
docker network rm my-network




# Показать все volumes
docker volume ls

# Создать volume
docker volume create my-volume

# Показать детали volume
docker volume inspect my-volume

# Удалить volume
docker volume rm my-volume

# Удалить неиспользуемые volumes
docker volume prune




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
