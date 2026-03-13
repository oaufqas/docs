**Docker** — это платформа для контейнеризации, которая упаковывает приложение со всеми зависимостями (библиотеками, настройками) в легкий, изолированный контейнер. 
Это обеспечивает одинаковую работу ПО на любых машинах, устраняя конфликты окружений. 

В отличие от виртуальных машин, Docker использует общее ядро ОС, что делает контейнеры быстрее и эффективнее.

## Основные понятия docker:
- #### [[./docker/containers.md|Containers]]
- #### [[./docker/images.md|Images]]
- #### [[./docker/volumes.md|Volumes]]
- #### [[./docker/networks.md|Networks]]
- #### [[./docker/docker_file|Dockerfile]], [[dockerfile-multi-stage|Multi-stage]], [[./docker/docker-compose|Docker-compose]]
- #### [[./docker/installation|Installation]], [[./docker/security|Security]]


**Docker Registry** — это централизованное хранилище (репозиторий) для ваших образов. Если провести аналогию с программированием, то Docker — это Git, а Registry — это GitHub для ваших образов.

### Виды реестров:

**[[./registry/dockerhub.md|Public]] (Публичные)**: Самый известный — Docker Hub. Там лежат официальные образы (Nginx, Python, Postgres), которые доступны всем.

**[[./registry/private-registry|Private]] (Приватные)**: Используются внутри компаний, чтобы посторонние не имели доступа к исходному коду и архитектуре приложения.

Cloud-решения: У каждого крупного провайдера есть свой сервис (например, Amazon ECR, Google Container Registry, Azure CR или Yandex Container Registry)