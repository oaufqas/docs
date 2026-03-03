# GitLab Runners: Полное руководство:

- Установка раннера
- Регистрация раннера
- Executor'ы: двигатели заданий
- Docker Executor в деталях
- Docker-in-Docker (dind) vs Docker Socket Binding
- Способы CD с GitLab Runner
- Теория работы раннера

### ----------------- Установка раннера --------------------

## 1. На сервер (нативно)

```bash
# 1. Добавляем официальный репозиторий GitLab Runner
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash

# 2. Устанавливаем раннер
sudo apt-get install gitlab-runner

# 3. Проверяем установку
gitlab-runner --version
```




## 2. В Docker-контейнер (рекомендуемый способ)
Раннер будет иметь доступ к сокету Docker, чтобы запускать другие контейнеры.


```bash
# Создаем директорию для конфига и запускаем контейнер
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```
 # Общий сокет! контейнеры будут запускаться РЯДОМ с контейнером runner'а


# Регистрация раннера

После установки раннер нужно зарегистрировать в GitLab, чтобы они могли общаться:
Зайди в GitLab → проект → Settings → CI/CD → Runners
Нажми "New project runner" (там будет инструкция с параметрами)


На **сервере**:
```bash
gitlab-runner register --url http://192.168.0.109 --token glrt-t3_NNzPzBDsu95ZZ5R5xfsX
```


В **Docker-контейнере**:
```bash
docker exec -it gitlab-runner gitlab-runner register --url http://192.168.0.109 --token glrt-t3_NNzPzBDsu95ZZ5R5xfsX

# При регистрации укажи: alpine:latest как базовый образ

nano /srv/gitlab-runner/config/config.toml
toml
[[runners]]
  name = "docker-runner"
  url = "https://gitlab.com/"
  token = "YOUR_TOKEN"
  executor = "docker"
  [runners.docker]
    image = "alpine:latest"  # Базовый образ для заданий
    privileged = true  # <<< Чтобы можно было использовать DinD
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]  # Проброс сокета
```

## Executor'ы: двигатели заданий
Когда Runner получает задание, ему нужно решить, где и как выполнять команды. За это отвечает компонент executor — стратегия создания изолированной среды для каждого задания.

# Executors:
1. Shell - Команды выполняются прямо на машине раннера в bash/powershell. Изоляция низкая (задания влияют друг на друга и хост). Простые скрипты, не требующие изоляции. Не для безопасности
2. Docker - Для каждого задания запускается новый Docker-контейнер. Изоляция высокая (каждое задание в чистом контейнере). Подавляющее большинство случаев. Чистота и повторяемость
3. Kubernetes - Для каждого задания создается новый Pod в кластере. Изоляция очень высокая. Масштабируемые окружения на Kubernetes
4. VirtualBox/Parallels - Запуск целой ВМ для задания. Изоляция полная (на уровне ОС). Тестирование под разные ОС (Windows, macOS, FreeBSD)




## Docker Executor в деталях:
1. Подготовка (Prepare)
- Runner общается с Docker-демоном, создает служебные контейнеры из секции services (postgres, redis и т.д.)

2. Пред-задание (Pre-job)
- Запускается служебный контейнер (Alpine Linux) для подготовки: клонирование кода, восстановление кэша, загрузка артефактов

3. Выполнение задания (Job)
- Запускается контейнер из образа, указанного в image. К нему подключается папка с кодом, выполняются команды из script

4. Пост-задание (Post-job)
- Снова запускается служебный контейнер для сохранения кэша и загрузки артефактов в GitLab




# Docker-in-Docker (dind) vs Docker Socket Binding
При работе с Docker внутри CI/CD есть два основных подхода: 



1. **Docker Socket Binding (рекомендуется)**
Как работает: Пробрасывает Docker-сокет с хоста в контейнер раннера. Контейнеры запускаются "рядом" (sibling containers) — на том же хосте.

volumes = ["/var/run/docker.sock:/var/run/docker.sock"] проброс при создании раннера

# Плюсы:
- Быстрее, нет вложенной виртуализации
- Разделяет кэш с хостом
- Проще в настройке
# Минусы:
- Меньше изоляции (есть доступ к демону хоста)
- Потенциальные проблемы с правами



2. **Docker-in-Docker (dind)**:
Внутри Docker контейнера (где работает твой job) нет Docker daemon'а, а значит нельзя выполнять docker build, docker push и другие Docker команды, НО м**ожно указать image: docker:latest**, и команды докера будут доступны раннеру. **DinD нужен для ИЗОЛИРОВАННОЙ СБОРКИ (docker login, build, push)**
DinD запускает полноценный Docker daemon внутри отдельного контейнера, к которому твой job может подключаться и использовать его для сборки образов.
Очень часто используется для сборки образов в ci

```yml
services:
  - docker:dind  # Запускает отдельный Docker-демон внутри контейнера
```

# Плюсы:
- Полная изоляция
- Чистое окружение для каждого задания
# Минусы:
- Медленнее (дополнительный слой)
- Проблемы с кэшированием
- Требует привилегированного режима


# Когда что использовать:
**Socket binding** — для большинства случаев, особенно когда нужна скорость
**DinD** — когда важна максимальная изоляция или нужно тестировать взаимодействие с разными версиями Docker






### Способы CD с GitLab Runner
**Способ 1:** Раннер на целевом сервере (executor = shell)
Раннер установлен прямо на сервере, куда деплоим. Он как локальный процесс выполняет команды.

```bash
##Проблемы и решения:

Права на запуск Docker:
sudo usermod -aG docker gitlab-runner

Права на sudo (если нужно):
sudo echo "gitlab-runner ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers


Проблемы с git и директориями:

# Вариант А: Добавить директорию в safe.directory
sudo -u gitlab-runner git config --global --add safe.directory /rent

# Вариант Б: Проверка и клонирование с раннером-владельцем
- |
  if [ ! -d "/rent/.git" ]; then
    echo "Directory not found. Initializing..."
    sudo mkdir -p /rent
    sudo chown gitlab-runner:gitlab-runner /rent
    git clone https://gitlab.com/project.git /rent
  fi
```




**Способ 2:** Удаленный деплой по SSH (executor = docker)
Раннер где угодно, подключается к серверу по SSH и выполняет команды удаленно.

```bash
Шаг 1: Подготовка ключей

# Создаем ключи (не на сервере, не в контейнере!, а на машине, с раннером)
ssh-keygen -t ed25519 -f gitlab_deploy_key -N ""

# Публичный ключ → на сервер в ~/.ssh/authorized_keys
# Приватный ключ → в переменные GitLab (Settings → CI/CD → Variables)

Шаг 2: Пайплайн с SSH

yaml
deploy:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
  script:
    - ssh -o StrictHostKeyChecking=no user@your-server "
        cd /path/to/project &&
        git pull &&
        docker-compose up -d --build
      "
  only:
    - main
```








### Теория работы раннера
- GitLab — заказчик
- Runner — работник, исполняющий команды из .gitlab-ci.yml

- Раннер может быть развернут в домашней локальной сети (на ВМ, где угодно). NAT роутера не помеха!
- Почему: Раннер и GitLab держат постоянное активное соединение. Не GitLab «стучится» к раннеру домой, а раннер сам подключается к GitLab. Он делает обычный исходящий запрос (как браузер), роутер через NAT выпускает трафик и знает, куда вернуть ответ.




## Best Practices

1. Установка: Docker-контейнер + executor = docker + базовый образ alpine:latest
2. Для сборки образов: Пробрасывай Docker-сокет (socket binding)
3. Для деплоя: Используй SSH-ключи (безопаснее, чем давать sudo)
4. Всегда указывай версии образов (не latest в production)
5. Используй кэширование для ускорения пайплайнов