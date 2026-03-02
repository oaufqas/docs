### Основные понятия:

- trigger - то, от чего срабатывает коммит
- variables(./variables.md) — глобальные переменные окружения, доступные во всех job’ах. 
- workflow — правила, определяющие, когда вообще создаётся pipeline.
- stages — последовательность этапов выполнения.
- default — значения по умолчанию для всех job’ов (чтобы не дублировать одно и то же).
- job’ы — конкретные задачи, которые выполняет CI/CD
- rules — решают появится ли job в pipeline вообще, а не просто когда он запустится.




## Stages — этапы pipeline
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




## Default — значения по умолчанию
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




## Jobs — определение задач
Job’ы — это основа всей конфигурации CI/CD. Каждый job описывает конкретную задачу, которую должен выполнить runner. Для job’а можно определить:
- Что выполнять (script). (обязательно!)
- На каком этапе (stage).
- На каком runner’е (tags).
- В каком окружении (image).
- Какие переменные использовать.
- Какие артефакты сохранить.
- Зависимости от других job’ов и многое другое.

```yml
stages:
  - build
  - test

build_app:
  stage: build
  image: node:18
  script:
    - npm install
    - npm run build
  artifacts:
    paths:
      - dist/    # build_app сохраняет результат сборки (dist/) как артефакт.
    expire_in: 1 day

run_tests:
  stage: test
  image: node:18
  script:
    - npm install
    - npm run test
  needs:
    - build_app  # Ждём выполнения build_app и получаем его артефакты:
                 # job не ждёт завершения всего stage build, а стартует сразу после build_app. Артефакты build_app будут автоматически доступны.
```




# Дополнительные параметры для job'ов

```yml
build_frontend:
  name: "Build Frontend Application" # С помощью name можно задать более понятное отображаемое название
  stage: build
  script:
    - npm run build

long_running_tests:
  stage: test
  timeout: 2h # timeout — максимальное время выполнения job’а (30s. 15m. 1h. 2h 30m.)
  script:
    - pytest --slow-tests

flaky_tests:
  stage: test
  retry:   # retry — автоматические повторы при ошибке. В примере повторяет до 2 раз, если упадёт
    max: 2
    when:  # Когда повторять
      - runner_system_failure
      - stuck_or_timeout_failure
  script:
    - npm test


tolerant_job: # Иногда ошибка допустима (например, необязательные тесты). Тогда можно явно сказать shell’у «игнорируй ошибку»:
  script:
    - npm test || true  # Даже если тесты упадут, job продолжит работу
  allow_failure: true # Альтернатива


build_job:
  stage: build
  variables: # Параметр variables позволяет задать переменные окружения только для конкретного job’а. Используются через $VAR или ${VAR}
    NODE_ENV: "production"
    LOG_LEVEL: "debug"
    CUSTOM_VAR: "some value"
  script:
    - echo $NODE_ENV      # Выведет: production
    - echo $LOG_LEVEL     # Выведет: debug

deploy_to_staging:
  stage: deploy
  environment: # Параметр environment связывает job с логическим окружением (staging, production и т. п.)
    name: staging
    url: https://staging.example.com
  script:
    - ./deploy-to-staging.sh

test_with_coverage:
  stage: test
  script:
    - pytest --cov=src --cov-report=term --cov-report=xml
  coverage: '/TOTAL.*\s+(\d+%)$/'  # Regex для извлечения % покрытия. GitLab может автоматически его извлечь и показать в интерфейсе.
```







## Workflow — условия запуска pipeline
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




### Rules - Базовый синтаксис
Ранее для этого использовались **only** и **except** — простые директивы, которые ограничивали запуск job’а по веткам, тегам или типу pipeline. Они работали, но были негибкими и плохо масштабировались, поэтому в современных конфигурациях **считаются устаревшими и заменены rules.**

Каждый элемент в rules — это правило, которое GitLab проверяет сверху вниз.

# Основные поля:
if — условие (логическое выражение, результат true или false).
when — что делать, если условие выполнилось.

# Чаще всего используются значения when:
always — job добавляется и запускается автоматически.
never — job не добавляется.
manual — job добавляется, но ждёт ручного запуска.
delayed — отложенный запуск (реже используется, разберём позже).

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

# Возможные значения $CI_PIPELINE_SOURCE(откуда произошел запуск pipline):

push — обычный git push.
merge_request_event — событие Merge Request.
schedule — запуск по расписанию.
web — ручной запуск из интерфейса.
api — запуск через API.
trigger — запуск из другого pipeline.


# Проверка наличия файлов (exists)

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

# Фильтрация по изменённым файлам (changes)
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





## image — Docker образ для выполнения job
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




# needs - зависимости между job’ами
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

# DAG вместо линейного pipeline, использование needs превращает pipeline в DAG (Directed Acyclic Graph) — граф зависимостей, совет:
Используйте artifacts для передачи результатов, а не для кэша.
Используйте needs, если хотите ускорить pipeline, но не переусердствуйте — слишком сложный DAG сложнее поддерживать.
Если job использует артефакты другого job’а — явно указывайте needs, даже если они находятся в соседних стадиях.