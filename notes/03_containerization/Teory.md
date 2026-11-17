**Docker** — это платформа для контейнеризации, которая упаковывает приложение со всеми зависимостями (библиотеками, настройками) в легкий, изолированный контейнер. 
Это обеспечивает одинаковую работу ПО на любых машинах, устраняя конфликты окружений. 

В отличие от виртуальных машин, Docker использует общее ядро ОС, что делает контейнеры быстрее и эффективнее. Основные понятия docker:
- #### [[notes/03_containerization/docker/containers|Containers]]
- #### [[images|Images]]
- #### [[notes/03_containerization/docker/volumes|Volumes]]
- #### [[notes/03_containerization/docker/networks|Networks]]
- #### [[docker_file|Dockerfile]], [[dockerfile-multi-stage|Multi-stage]], [[docker-compose|Docker-compose]]
- #### [[notes/03_containerization/docker/installation|Installation]], [[notes/03_containerization/docker/security|Security]]


**Docker Registry** — это централизованное хранилище (репозиторий) для ваших образов. Если провести аналогию с программированием, то Docker — это Git, а Registry — это GitHub для ваших образов.

#### Виды реестров:

- **[[dockerhub|Public]] (Публичные)**: Самый известный — Docker Hub. Там лежат официальные образы (Nginx, Python, Postgres), которые доступны всем.
- **[[private-registry|Private]] (Приватные)**: Используются внутри компаний, чтобы посторонние не имели доступа к исходному коду и архитектуре приложения.
- Cloud-решения: У каждого крупного провайдера есть свой сервис (например, Amazon ECR, Google Container Registry, Azure CR или Yandex Container Registry)


#### Ошибки при установке или запуске контейнеров:

1. Остановите Docker и containerd:

```bash
sudo systemctl stop docker
sudo systemctl stop containerd
```

2. Удалите директории с данными и временными файлами:  

Это принудительно очистит базу данных снимков (`metadata.db`), на которую ругается ошибка.

```bash
sudo rm -rf /var/lib/containerd
sudo rm -rf /run/containerd
```

Примечание: это удалит все скачанные образы и созданные контейнеры.

3. Сброс конфигурации containerd (опционально)

Если ошибка сохраняется, возможно, в системе остался старый или некорректный файл конфигурации. Его можно пересоздать с настройками по умолчанию:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

4. Перезапуск сервисов

После очистки директорий запустите сервисы заново. Containerd автоматически пересоздаст нужную структуру папок и чистую базу данных `metadata.db` при первом обращении. 

```bash
sudo systemctl start containerd
sudo systemctl start docker
```