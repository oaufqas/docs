**Docker** — это платформа для контейнеризации, которая упаковывает приложение со всеми зависимостями (библиотеками, настройками) в легкий, изолированный контейнер. 
Это обеспечивает одинаковую работу ПО на любых машинах, устраняя конфликты окружений. 
В отличие от виртуальных машин, Docker использует общее ядро ОС, что делает контейнеры быстрее и эффективнее.

## Основные понятия docker:
containers(./docker/containers.md)
images(./docker/images.md)
volumes(./docker/volumes.md)
networks(./docker/networks.md)
dockerfile(./docker/dockerfile.md)
docker-compose(./docker/docker-compose.md)

**Docker Registry** — это централизованное хранилище (репозиторий) для ваших образов. Если провести аналогию с программированием, то Docker — это Git, а Registry — это GitHub для ваших образов.

### Виды реестров:
Public(./registry/dockerhub.md) (Публичные): Самый известный — Docker Hub. Там лежат официальные образы (Nginx, Python, Postgres), которые доступны всем [2].
Private(./registry/private-registry.md) (Приватные): Используются внутри компаний, чтобы посторонние не имели доступа к исходному коду и архитектуре приложения.
Cloud-решения: У каждого крупного провайдера есть свой сервис (например, Amazon ECR, Google Container Registry, Azure CR или Yandex Container Registry)