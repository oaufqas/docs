### Структура Job'а и его параметры:

Job — это отдельная задача внутри pipeline. Именно job’ы выполняют реальную работу: запускают тесты, собирают проект, деплоят код и т. д.

Строго говоря, единственный действительно обязательный параметр job’а — это script. Всё остальное либо имеет значение по умолчанию, либо наследуется.

Команды выполняются строго по порядку.Каждая команда запускается в одной и той же среде (контейнере). Если любая команда возвращает ненулевой код выхода, job считается failed, и выполнение останавливается.


```yml
failing_job:
  script:
    - echo "This will print"
    - false                    # Эта команда вернёт код ошибки
    - echo "This won't print"  # До сюда не дойдёт
```

##### Почему `.full-deploy` может быть с точкой?

В GitLab CI `.` в начале имени означает **скрытый job** — он не будет выполняться как отдельная задача, только как [[pipline_syntax#Якоря (anchors) и алиасы (aliases)|шаблон]].

```yaml
.full-deploy: &full-deploy  # С точкой = скрытый, только для шаблонов
  stage: deploy
  
  
visible-job:  # Без точки = будет выполняться
  <<: *full-deploy
  script: echo "Я видим!"
```


---
### Jobs — определение задач:

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


---

#### Дополнительные параметры для job'ов

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