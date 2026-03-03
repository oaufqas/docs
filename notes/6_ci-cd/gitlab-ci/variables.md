### ------------------------------- Variables ----------------------------------

- GitLab предоставляет большой набор встроенных CI-переменных, которые автоматически доступны в каждом job’е: CI_COMMIT_SHA, CI_PROJECT_ID, CI_PIPELINE_ID, CI_MERGE_REQUEST_ID и многие другие.
- Для секретов и чувствительных данных используется раздел **Settings → CI/CD → Variables в интерфейсе GitLab**, там вы можете:

1. Создавать переменные окружения.
2. Помечать их как protected (доступны только для protected-веток и тегов).
3. Помечать как masked (значения не выводятся в логах).
4. Это стандартный и безопасный способ хранения токенов, ключей и паролей.
5. Задать переменные, привязанные к конкретным environment’ам (например, разные значения для staging и production).


## Variables — глобальные переменные
Эти переменные автоматически подставляются в окружение каждого job’а и могут использоваться в скриптах.
Важно: значения из variables хранятся прямо в репозитории и видны всем, кто имеет доступ к .gitlab-ci.yml.

```yml
variables:
  LOG_LEVEL: "info"
  DATABASE_URL: "postgres://localhost/mydb"
  DOCKER_REGISTRY: "registry.example.com"
  ENVIRONMENT_NAME: "staging
```



# Приоритет переменных (от низшего к высшему):
- Встроенные переменные GitLab (CI_COMMIT_SHA, CI_PROJECT_ID и т.д.).
- Переменные из Settings → CI/CD → Variables.
- Глобальные переменные (variables в .gitlab-ci.yml).
- Переменные из default.
- Переменные, объявленные непосредственно в job’е.



# Основные группы и самые важные переменные, которые есть по умолчанию в GitLab:

1. Информация о пайплайне и задаче (Job)
- $CI_PIPELINE_ID: **Уникальный ID текущего пайплайна.**
- $CI_PIPELINE_SOURCE: **Откуда пришел запрос** (push, web, schedule, merge_request_event). 
- $CI_JOB_ID: **Уникальный ID текущей задачи.**
- $CI_JOB_NAME: **Название задачи из вашего .gitlab-ci.yml.**
- $CI_JOB_STAGE: **Название стадии (stage) задачи.**
- $CI_JOB_TOKEN: **Временный токен для авторизации в API GitLab или Docker Registry из скрипта.**

2. Информация о коммите и ветке
- $CI_COMMIT_SHA: **Полный хеш коммита.** (abc123def456...)
- $CI_COMMIT_SHORT_SHA: **Первые 8 символов хеша.**
- $CI_COMMIT_REF_NAME: **Название ветки или тега, для которого запущен пайплайн.**
- $CI_COMMIT_BRANCH: **Название ветки (доступно только в пайплайнах для веток).** (main, develop, feature/new-ui)
- $CI_COMMIT_TAG: **Название тега (если это tag pipeline).** (v1.0.0, release-2025-12-01)
- $CI_COMMIT_MESSAGE: **Полный текст сообщения коммита.** (Fix: update dependencies)
- $CI_COMMIT_TITLE: **Только первая строка сообщения коммита.**

3. Доступны только в pipeline, связанных с Merge Request:
- $CI_MERGE_REQUEST_IID: **ID Merge Request (в рамках проекта)** 42
- $CI_MERGE_REQUEST_TARGET_BRANCH_NAME: **Целевая ветка Merge Request** main
- $CI_MERGE_REQUEST_DRAFT: **Является ли MR черновиком (draft)** true, false

4. Информация о проекте
- $CI_PROJECT_ID: **Уникальный ID проекта в GitLab.**
- $CI_PROJECT_NAME: **Имя проекта.** my-awesome-app
- $CI_PROJECT_PATH: **Путь к проекту (например, group/subgroup/project).** group/subgroup/my-project
- $CI_PROJECT_DIR: **Полный путь, по которому склонирован репозиторий внутри раннера.**
- $CI_REPOSITORY_URL: **URL для клонирования репозитория с токеном доступа.**

5. Реестр контейнеров (Container Registry) Если вы используете встроенный GitLab Registry:
- $CI_REGISTRY: **Адрес реестра (обычно registry.gitlab.com).**
- $CI_REGISTRY_IMAGE: **Базовый адрес образа для вашего проекта.**
- $CI_REGISTRY_USER: **Имя пользователя для логина (обычно gitlab-ci-token).**
- $CI_REGISTRY_PASSWORD: **Пароль для логина (то же самое, что $CI_JOB_TOKEN).**

6. Окружение (Environment)
- $CI_ENVIRONMENT_NAME: **Имя окружения (например, production или staging).**
- $CI_ENVIRONMENT_URL: **URL этого окружения.**

- $CI_DEBUG_TRACE: **отладка условий и выполнения job’ов** true/false 
GitLab выводит подробный shell-лог выполнения job.