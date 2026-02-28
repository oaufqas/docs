### ------------------- installation runner ------------------------

# direct installation on the server


# 1. Добавляем официальный репозиторий GitLab Runner
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash

# 2. Устанавливаем раннер
sudo apt-get install gitlab-runner

# 3. Проверяем, что установка прошла успешно
gitlab-runner --version



### --------------------------------------------------



# installation runner in docker container
Раннер будет иметь доступ к сокету docker'a чтобы запускать другие контейнеры 

# Создаем директорию для конфига и запускаем контейнер
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest


# Регистрация раннера 
После установки раннера, его нужно зарегестрировать в gitlab, чтобы они смогли общаться:
- Зайди в свой GitLab, открой проект, для которого нужен раннер.
- Перейди: Settings → CI/CD → Runners.
- Нажми "New project runner" . Там же будет инструкция с нужными параметрами.

## Регистрация раннера локального на сервере
gitlab-runner register  --url http://192.168.0.109  --token glrt-t3_NNzPzBDsu95ZZ5R5xfsX

## Регистрация раннера из docker container
docker exec -it gitlab-runner gitlab-runner register  --url http://192.168.0.109  --token glrt-t3_NNzPzBDsu95ZZ5R5xfsX

alpine:latest

Изменить конфиг

# nano /srv/gitlab-runner/config/config.toml
[[runners]]
  name = "docker-runner"
  url = "https://gitlab.com/"
  token = "YOUR_TOKEN"
  executor = "docker"
  [runners.docker]
    image = "alpine:latest"  # Базовый образ для заданий
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"] <-------------




### --------------------- Теория ------------------------------------------

Лучшая практика: установка раннера из docker container + выбор executor docker, default image - alpine:latest

Gitlab - заказчик, runner - работник, исполняющий заказы, в нашем случае команды из .gitlab-ci.yml

Когда Runner получает задание. Ему нужно решить, где и как выполнять команды вроде npm test или docker build. За это отвечает компонент под названием executor. Executor — это стратегия или "движок", который создает изолированную среду для каждого задания. Ты выбираешь executor при регистрации раннера. **Вот основные варианты:**


- Shell - Команды выполняются прямо на машине, где установлен Runner, в обычной командной оболочке (bash, powershell) . Уровень изоляции низкий (задания влияют друг на друга и на хост). Самые простые скрипты, когда не хочется заморачиваться с контейнерами. Не подходит для безопасности, если проекты не доверяют друг другу.

- Docker - Для каждого задания Runner запускает новый Docker-контейнер и выполняет команды внутри него. Уровень изоляции высокий (каждое задание в своем чистом контейнере). Подавляющее большинство случаев. Гарантирует чистоту и повторяемость сборок.

- Kubernetes - Для каждого задания Runner создает новый Pod в кластере Kubernetes. Уровень изоляции очень высокий.Масштабируемые окружения, где инфраструктура уже построена на Kubernetes.

- VirtualBox/Parallels. Запускает целую виртуальную машину для задания. Уровень изоляции полный (изоляция на уровне ОС). Специфические случаи, когда нужно тестировать под разные ОС (Windows, macOS, FreeBSD).




## Как работает Docker Executor

- Runner общается с Docker-демоном на своей машине. Он создает и запускает все необходимые служебные контейнеры, которые ты указал в секции services (например, postgres:latest или redis:latest)

- Запускается специальный служебный контейнер (на базе Alpine Linux). Его задача — подготовить рабочее пространство: склонировать код из репозитория, восстановить кэш и скачать артефакты из предыдущих стадий.

- Выполнение задания (Job). Это главный акт. Runner запускает контейнер из образа, который ты указал в директиве image (например, node:18). Он подключает к этому контейнеру папку с кодом, подготовленную на предыдущем шаге, и выполняет все команды из script.

- Пост-задание (Post-job). Снова запускается служебный контейнер, чтобы сохранить кэш и загрузить артебефекты обратно в GitLab




### ------------------------ Использование раннера в деплое ------------------------------

**Раннер может быть развернут в домашней локальной сети**, (на виртуальной машине, где угодно), и nat адресация роутера не помешает ему, как может показаться на первый взгяд. **Раннер и гитлаб держат постоянное активное соединение:** Не GitLab «стучится» к вашему раннеру домой, а раннер сам подключается к GitLab. Он делает обычный исходящий запрос (как браузер, когда вы открываете сайт). Ваш роутер через NAT спокойно выпускает этот трафик наружу и знает, куда вернуть ответ.







## Есть 2 распространенных способа CD с помощью gitlab runner'a:



- **Установка раннера на целевой сервер** (на который будем деплоить), и ему, как процессу на сервере описать что нужно делать

При деплое этим способом, столкнулся с несколькими проблемами:

Чтобы адекватно запускать программы нужно gitlab-runner'у давать либо права sudo либо назначать овнером директории, если гит клон сделать с другого юзера, то гит будет ругаться при подтягивании изменений раннером, это нужно предусматривать в пайплайне:



# executor SHELL!!!!!
```bash
sudo -u gitlab-runner git config --global --add safe.directory /rent
```
Либо сделать проверку в пайплайне на наличие папки, и клонировать уже с раннера:
```bash
   - |
      if [ ! -d "/rent/.git" ]; then
        echo "Directory /rent not found or not a git repo. Initializing..."
        sudo mkdir -p /rent
        sudo chown gitlab-runner:gitlab-runner /rent
        git clone https://gitlab.com/oaufqas/rent.git /rent
      fi
```
но при использовании докера, нужно добавить раннер в его группу: **sudo usermod -aG docker gitlab-runner**
также для использования sudo в пайплайне нужно изменть настройки sudoers: **sudo echo "gitlab-runner   ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers**





- **Развертывание удаленно по SSH**, на сервер мы закидываем публичный ключ раннера, описываем ему как подключиться к серверу и что делать удаленно:

Шаг 1: Подготовка ключей
- Если у вас еще нет ключей, создайте их на локальной машине (не на сервере и не в контейнере):
# ssh-keygen -t ed25519 -f gitlab_deploy_key -N ""
- Публичный ключ (gitlab_deploy_key.pub): скопируйте и добавьте на удаленный сервер в файл ~/.ssh/authorized_keys.
- Приватный ключ (gitlab_deploy_key): это секрет, который мы положим в GitLab


```yaml
deploy:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client  # Устанавливаем SSH клиент
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | ssh-add -  # Добавляем приватный ключ
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