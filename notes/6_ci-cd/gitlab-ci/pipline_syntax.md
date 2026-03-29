### Основные понятия:

- trigger - то, от чего срабатывает коммит
- variables(./variables.md) — глобальные переменные окружения, доступные во всех job’ах. 
- workflow — правила, определяющие, когда вообще создаётся pipeline.
- stages — последовательность этапов выполнения.
- default — значения по умолчанию для всех job’ов (чтобы не дублировать одно и то же).
- job’ы — конкретные задачи, которые выполняет CI/CD
- rules — решают появится ли job в pipeline вообще, а не просто когда он запустится.

---
#### Stages — этапы pipeline
Stage (этап) — это логическая группа job’ов, которые могут выполняться параллельно.
Сами stage’и при этом выполняются строго последовательно.

```yml
stages:
  - test
  - build
  - deploy
```

Такое описание означает следующее:
- Сначала запускаются все job’ы с stage: test.
- Если все они завершились успешно, запускаются job’ы с stage: build.
- После успешного build начинается deploy.


---

#### Default — значения по умолчанию

Чтобы не дублировать одни и те же параметры в каждом job’е, в GitLab CI существует секция default. Все job’ы наследуют значения, указанные в этом блоке, если не переопределяют их явно.
Обратите внимание: если параметр указан и в default, и в job’е — всегда выигрывает значение из job’а.

```yml
default:
  image: python:3.11
  timeout: 1h
  retry: 1
  tags:
    - docker

build_job:
  stage: build
  script:
    - echo "This job uses Python 3.11 image by default"

test_job:
  stage: test
  script:
    - echo "This job also uses Python 3.11 image"
  timeout: 30m  # Переопределяем timeout только для этого job'а
```

---

#### Workflow — условия запуска pipeline

Секция workflow отвечает за самое первое решение: нужно ли вообще создавать pipeline.
Это уровень выше, чем условия у отдельных job’ов — если pipeline не создан, никакие job’ы даже не будут рассмотрены.

```yml
workflow:
  rules:
    # Не запускаем pipeline, если в сообщении коммита есть [skip ci]
    - if: '$CI_COMMIT_MESSAGE =~ /\[skip ci\]/'
    # Запускаем pipeline для merge request'ов
    - if: '$CI_PIPELINE_SOURCE  "merge_request_event"'
    # Запускаем для push'ей в ветку main
    - if: '$CI_COMMIT_BRANCH  "main"'
    # Запускаем для тегов
    - if: '$CI_COMMIT_TAG'
    # Во всех остальных случаях pipeline не создаём
    - when: never
```

---

#### Rules - Базовый синтаксис:

Ранее для этого использовались **only** и **except** — простые директивы, которые ограничивали запуск job’а по веткам, тегам или типу pipeline. Они работали, но были негибкими и плохо масштабировались, поэтому в современных конфигурациях **считаются устаревшими и заменены rules.**

Каждый элемент в rules — это правило, которое GitLab проверяет сверху вниз.

#### Основные поля rules:
`if` — условие (логическое выражение, результат true или false).
`when` — что делать, если условие выполнилось.

#### Чаще всего используются значения when:
`always` — job добавляется и запускается автоматически.
`never` — job не добавляется.
`manual` — job добавляется, но ждёт ручного запуска.
`delayed` — отложенный запуск (реже используется, разберём позже).

```yml
test_main_only:
  stage: test
  script:
    - npm test
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"' # Если ветка не main, второе правило гарантирует, что job не появится в pipeline.
      when: always
    - when: never

build_main:
  stage: build
  script: 
    - echo "Сборка main"
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH' # Вместо "main" используйте $CI_DEFAULT_BRANCH, чтобы конфиг был переносимым между проектами:
      when: always
    - when: never
release_build:
  stage: build
  script:
    - npm run build
  rules:
    - if: '$CI_COMMIT_TAG' # $CI_COMMIT_TAG содержит имя тега или пустую строку. Если pipeline запущен не из тега, условие просто не выполнится.
      when: always
```

##### Возможные значения $CI_PIPELINE_SOURCE(откуда произошел запуск pipline):

`push` — обычный git push.
`merge_request_event` — событие Merge Request.
`schedule` — запуск по расписанию.
`web` — ручной запуск из интерфейса.
`api` — запуск через API.
`trigger` — запуск из другого pipeline.

---
#### Проверка наличия файлов (exists)

```yml
build_only_on_src_change:
  stage: build
  script:
    - npm run build
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      exists: # exists проверяет, есть ли файлы в репозитории. Это полезно, если структура проекта может отличаться, например в монорепозитории.
        - src/**/*
      when: always
    - when: never
```


#### Фильтрация по изменённым файлам (changes)

Один из самых эффективных способов сократить время pipeline в больших проектах — запуск job только если изменились нужные файлы:

```yml
test_backend:
  stage: test
  script:
    - cd backend && pytest
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      changes: # changes проверяет дифф коммита, а не наличие файлов.
        - backend/**/*
        - shared/**/*
      when: always
    - when: never
```


---

#### image — Docker образ для выполнения job

GitLab CI чаще всего работает в Docker-контейнерах. Параметр image определяет, в каком окружении будут выполняться команды из script.
Если image указан в job’е — используется он. Если нет — берётся image из секции default. Если image не указан нигде — используется образ по умолчанию GitLab Runner (что часто приводит к неожиданностям).

```yml
default:
  image: ubuntu:22.04

python_job:
  stage: build
  image: python:3.11
  script:
    - python --version
    - pip install -r requirements.txt

node_job:
  stage: build
  image: node:18
  script:
    - node --version
    - npm install
```

---

#### needs - зависимости между job’ами

По умолчанию GitLab выполняет pipeline строго по стадиям: все job’ы стадии build → потом test → потом deploy.
Параметр **needs** позволяет задать явные зависимости между job’ами и **тем самым ускорить pipeline**.
Артефакты автоматически передаются между зависимыми job’ами.

```yml
stages:
  - build
  - test
  - package

build_app:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/

test_app: # test_app не ждёт завершения всей стадии build, ему нужен только build_app
  stage: test
  needs:
    - build_app  # Ждём артефакты от build_app
  script:
    - npm test
  artifacts:
    paths:
      - test-results/

package_app:
  stage: package
  needs:
    - build_app     # Можем зависеть от нескольких job'ов
    - test_app
  script:
    - ./package.sh
```

###### DAG вместо линейного pipeline, использование needs превращает pipeline в DAG (Directed Acyclic Graph) — граф зависимостей, совет:

- Используйте artifacts для передачи результатов, а не для кэша.
- Используйте needs, если хотите ускорить pipeline, но не переусердствуйте — слишком сложный DAG сложнее поддерживать.
- Если job использует артефакты другого job’а — явно указывайте needs, даже если они находятся в соседних стадиях.

---

#### Services - вспомогательные контейнеры

Иногда для выполнения job’а одного контейнера недостаточно. Например, для тестов может понадобиться база данных, Redis, RabbitMQ или другой внешний сервис.
В GitLab CI для этого есть services — **дополнительные Docker-контейнеры, которые запускаются рядом с основным job’ом.**

```yml
test_with_db:
  stage: test
  image: python:3.11
  services:
    - postgres:17 # <- Сервис-зависимость база данных
    - redis:7 # <- Сервис-зависимость redis
  variables: 
    POSTGRES_DB: test_db 
    POSTGRES_USER: test_user
    POSTGRES_PASSWORD: test_password
    DATABASE_URL: "postgresql://test_user:test_password@postgres:5432/test_db"
    REDIS_URL: "redis://redis:6379"
  script:
    - pip install -r requirements.txt
    - pytest
```

Обращение к сервисам происходит:
- К PostgreSQL — по хосту postgres.
- К Redis — по хосту redis.

Алиасы сервисов
Иногда удобнее использовать более осмысленные имена. Для этого можно задать alias:
```yml
  services:
    - name: postgres:17
      alias: db # <- Теперь база доступна по хосту db, а не postgres
```

---

#### Готовность сервиса — важный нюанс

Сервисы не гарантированно готовы к моменту старта script. Частая ошибка новичков — сразу подключаться к БД и ловить connection refused.
GitLab не ждёт, пока сервис полностью поднимется.
#### Обычно используют один из подходов:
- retry-логику в коде.
- sleep (не лучший вариант).
- Утилиты ожидания (wait-for-it, pg_isready, nc и т.п.).
#### Когда стоит использовать services:
- unit и integration-тестов.
- Локальных БД для CI.
- Проверки миграций.
- Временных очередей и кэшей.
#### Не стоит использовать их для:
- production-нагрузок.
- Долгоживущих окружений.
- Сложных multi-container сценариев (для этого лучше Docker Compose или Kubernetes).

---

#### before_script and after_script:
##### before:
- Установка зависимостей.
- Подготовка окружения.
- Логин в registry.
- Проверка доступности сервисов.
##### after:
- Очистка временных файлов.
- Логирование.
- Отладочная информация.
- Уведомления.


```yml
build_job:
  stage: build
  before_script:
    - echo "Preparing environment..."
    - npm install
  script:
    - npm run build
  after_script:
    - echo "Cleaning up..."
    - rm -rf node_modules
```


---

### Матрица параллельных job’ов (parallel: matrix)

Матрицы позволяют запускать один job с разными комбинациями переменных.

```yml
test_matrix:
  stage: test
  image: python:3.9
  parallel:
    matrix:
      - PYTHON_VERSION: ["3.8", "3.9", "3.10", "3.11"]
        DJANGO_VERSION: ["3.2", "4.0"]
  script:
    - echo "Testing Python $PYTHON_VERSION with Django $DJANGO_VERSION"
    - pip install django==$DJANGO_VERSION
    - pytest
```


#### Что произойдёт:
- GitLab создаст 8 отдельных job’ов (4 × 2).
- Каждый job получит свою комбинацию переменных.
- В UI они будут отображаться как отдельные задания.

#### Это особенно удобно для:
- Тестирования на нескольких версиях языка.
- Проверки совместимости библиотек.
- Кросс-платформенных сборок.

#### Важные ограничения матриц:
- Количество job’ов в матрице ограничено (зависит от версии GitLab).
- Большое количество комбинаций может резко увеличить время pipeline’а.
- Каждый job — это отдельный контейнер со своими ресурсами.

Поэтому матрицы стоит использовать осознанно и не превращать pipeline в «взрыв job’ов».


---

###  Якоря (anchors) и алиасы (aliases)

Это функции в yml, например у тебя есть **повторяющийся кусок кода**, и ты не хочешь копировать его 10 раз. Ты делаешь **шаблон** (anchor), а потом **ссылаешься** на него (alias).

##### 1. **Создание шаблона (Anchor)** — `&имя`

```yaml
.full-deploy: &full-deploy
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache ansible openssh-client
```

`&full-deploy` — это якорь. Он "запоминает" этот блок.

##### 2. **Использование шаблона (Alias)** — `<<: *имя`

```yaml

deploy-app:
  <<: *full-deploy  # Вставь сюда всё, что в &full-deploy
  script:
    - echo "Деплою..."
```

`<<: *full-deploy` означает: "возьми всё из якоря full-deploy и вставь сюда".
