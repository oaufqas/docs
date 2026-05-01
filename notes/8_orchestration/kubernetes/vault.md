# Vault

##### HashiCorp Vault - это мощный инструмент для безопасного управления секретами. Если кратко: это «сейф» для ваших паролей, сертификатов и ключей, к которому приложения обращаются по API. 

В обычном подходе секреты часто лежат в переменных окружения или Git-репозиториях (что небезопасно). Vault решает эту проблему.

### Установка через yandex зеркало [[helm#Установка HashiCorp-Vault|(Офиц репозиторий)]]:

```bash
helm pull oci://cr.yandex/yc-marketplace/yandex-cloud/vault/chart/vault \ 
--version 0.28.1+yckms \ 
--untar
```

### DEV Mode

```bash
helm install vault ./vault \ 
--set "server.dev.enabled=true" \ 
--namespace vault \ 
--create-namespace
```

### Prod Mode

```yaml
server:
  dev:
    enabled: false
    
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
      config: |
        ui = true
        listener "tcp" {
          tls_disable = 1
          address = "[::]:8200"
          cluster_address = "[::]:8201"
        }
        storage "raft" {
          path = "/vault/data"
        }

  image:
    repository: "mirror.gcr.io/hashicorp/vault"
    tag: "1.15.0"
    # repository: "cr.yandex/mirror/hashicorp/vault" tag: "latest

  # Настройка постоянного диска (в Minikube создастся автоматически)
  dataStorage:
    enabled: true
    size: 10Gi
```

```bash
helm install vault -f values.yaml ./vault -n vault --create-namespace


# Заходим в под с vault
kubectl exec -it vault-0 -- sh

vault status # Вывод должен быть:
# Key                     Value
# ---                     -----
# Seal Type               shamir
# Initialized             false
# Sealed                  true
# ...

vault operator init --key-shares=<на сколько разделить> --key-threshold=<сколько кусков нужно>

vault operator unseal # <key-threshold> раз с разными ключами
```


---

### Настройка Vault для работы в Kubernetes

```bash
# 1. Включаем метод авторизации
vault auth enable kubernetes

# 2. Настраиваем связь с API кластера
# Внутри пода Vault переменные $KUBERNETES_... доступны автоматически
vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"
```

Создание секретов в разных «папках», разделим данные физически по путям:

```bash
# Для сервера
vault kv put secret/rent/server db_password="app_pass" api_key="abc"

# Для базы данных
vault kv put secret/rent/db root_password="root_pass"
```

Написание политики (Policies) — кто что видит:

**Файл `server-policy.hcl`:**

```hcl
path "secret/data/app/server" {
  capabilities = ["read"]
}
```

**Файл `db-policy.hcl`:**

```hcl
path "secret/data/rent/db" {
  capabilities = ["read"]
}
```

Загружаем их в Vault:  
`vault policy write server-policy server-policy.hcl`  
`vault policy write db-policy db-policy.hcl`

Создание ролей (Roles) — связь пода и политики

Роль говорит: «Если пришел под с таким-то ServiceAccount, дай ему такую-то политику».

```bash
# Роль для сервера
vault write auth/kubernetes/role/server-role \
    bound_service_account_names=server-sa \
    bound_service_account_namespaces=default \
    policies=server-policy \
    ttl=1h

# Роль для базы
vault write auth/kubernetes/role/db-role \
    bound_service_account_names=db-sa \
    bound_service_account_namespaces=default \
    policies=db-policy \
    ttl=1h
```


Настройка на стороне Kubernetes (Инъектор)

Чтобы секреты превратились в **переменные окружения**, в манифестах `Deployment` (через Helm) нужно добавить аннотации-шаблоны.

**Для сервера (пример):**

```yaml
annotations:
  ://hashicorp.com: "true"
  ://hashicorp.com: "server-role" # Имя роли из шага 4
  ://hashicorp.com-secret-env: "secret/data/rent/server"
  # Шаблон, который превращает секрет в формат export VAR=VAL
  ://hashicorp.com-template-env: |
    {{- with secret "secret/data/rent/server" -}}
    {{- range $k, $v := .Data.data }}
    export {{ $k }}="{{ $v }}"
    {{- end }}
    {{- end -}}
```

Запуск приложения

Поскольку Vault Agent кладет эти `export` в файл `/vault/secrets/env`, твое приложение должно их «подхватить» при старте. В `command` контейнера нужно прописать:  
`command: ["/bin/sh", "-c", "source /vault/secrets/env && node index.js"]`


---

### Основная информация про vault


1. **Централизованное хранение**: Все секреты (пароли от БД, API-ключи) хранятся в одном месте в зашифрованном виде.
2. **Динамические секреты**: Vault может генерировать пароли «на лету». Например, когда приложению нужно подключиться к PostgreSQL, Vault создает временного пользователя в базе, выдает пароль приложению, а через час сам его удаляет.
3. **Управление сертификатами (PKI)**: Vault может работать как ваш внутренний центр сертификации, выдавая SSL/TLS сертификаты.
4. **Шифрование как сервис (Transit)**: Вы можете отправлять данные в Vault, он их зашифрует и вернет обратно. Вашему приложению не нужно самому хранить ключи шифрования.

Как это работает в Kubernetes 

Обычно в K8s используют **Vault Agent Injector**. Процесс выглядит так:

1. Vault Server (Сервер)

Это **центральное хранилище** и «мозги» всей системы. В твоем кластере это обычно Pod с именем `vault-0`.

- **Что делает:** Хранит данные (в зашифрованном виде), проверяет права доступа (Policy), авторизует пользователей и выдает токены.
- **Как работает:** К нему обращаются через API (порт 8200). Он ничего не знает о твоих подах, пока они сами к нему не придут с запросом.
- **Аналогия:** Это бронированный сейф в банке с охранником, который проверяет документы.

2. Vault Agent Injector (Инъектор)

Это вспомогательный сервис, который «подселяет» секреты в твои приложения. В кластере это отдельный Pod (обычно `vault-agent-injector`).

- **Что делает:** Он следит за созданием новых подов в кластере. Если он видит под с особыми пометками (**аннотациями**), он автоматически меняет его структуру «на лету».
- **Как работает (магия инъекции):**
    1. Ты отправляешь манифест `Deployment` в K8s.
    2. Инъектор перехватывает его.
    3. Он добавляет в твой под дополнительный маленький контейнер — **Vault Agent** (sidecar-контейнер).
    4. Этот агент сам идет к **Серверу**, забирает секреты и кладет их в общую память пода (папка `/vault/secrets/`).

---

Архитектурные особенности, которые важно знать

- **Распечатывание (Unsealing)**: Когда Vault запускается (не в режиме `dev`), он находится в состоянии «запечатано». Он зашифрован ключом, который разбит на несколько частей (обычно 5). Чтобы сейф открылся, нужно ввести минимум 3 части ключа (алгоритм Шамира).
- Ключевые параметры при инициализации

Ты можешь изменить количество ключей при запуске:  
`vault operator init --key-shares=7 --key-threshold=4`

- **Key Shares (Доли):** На сколько частей разрезать ключ (в примере — 7).
- **Key Threshold (Порог):** Сколько частей нужно собрать вместе, чтобы открыть Vault (в примере — 4).

![[Pasted image 20260428232610.png]]

- **Пути (Paths)**: Секреты организованы как файловая система. Например: `secret/data/frontend/api-keys`.
- **Политики (Policies)**: Вы можете четко прописать: «Под фронтенда может только читать из папки A, а админ может всё».

![[Pasted image 20260428230547.png]]

### Основные команды

1. Статус и авторизация

Прежде чем что-то делать, нужно понять, «открыт» ли сейф и кто вы в нём.

- `vault status` — Проверка состояния. Главное поле: **Sealed** (`true` или `false`). Если `true`, Vault зашифрован и не отдаст данные.
- `vault login <token>` — Вход в Vault по токену. В dev-режиме это обычно `root`.
- `vault operator init` — Инициализация сейфа секретов, выдаст 5 ключей по умолчанию.
- `vault operator unseal` — Ввод ключа на "распечатывание" сейфа vault, при `--key-threshold` > 1, нужно будет вызвать команду несколько раз с разными ключами.
---

2. Работа с секретами (KV — Key/Value)

Это основное хранилище для паролей. Мы используем версию **v2**, которая поддерживает версионность (можно посмотреть старые пароли).

- `vault secrets enable -path=my-secrets kv-v2` — Включить «движок» для хранения секретов по пути `my-secrets/`.
- `vault kv put my-secrets/db-config user="admin" pass="12345"` — **Записать** секрет. Если по этому пути уже что-то было, Vault создаст новую версию.
- `vault kv get my-secrets/db-config` — **Прочитать** текущую (последнюю) версию секрета.
- `vault kv get -version=2 my-secrets/db-config` — Прочитать конкретную старую версию.
- `vault kv list my-secrets/` — Показать все ключи (папки), которые лежат по этому пути.
- `vault kv delete my-secrets/db-config` — Удалить последнюю версию (её можно восстановить).
- `vault kv metadata delete my-secrets/db-config` — Удалить секрет **навсегда** вместе со всей историей.

---

3. Политики (Policies)

Политики определяют, кто и куда имеет доступ. Это обычные текстовые файлы `.hcl`.

- `vault policy list` — Список всех созданных политик.
- `vault policy read my-policy` — Посмотреть содержимое конкретной политики.
- `vault policy write my-policy ./policy-file.hcl` — Создать или обновить политику из файла.

---

4. Авторизация (Auth Methods)

Способы, которыми пользователи или приложения доказывают, что они — это они.

- `vault auth list` — Посмотреть включенные методы (token, userpass, kubernetes и т.д.).
- `vault auth enable kubernetes` — Включить проверку через токены Kubernetes.
- `vault write auth/kubernetes/role/app-role ...` — Создать «роль», которая связывает ServiceAccount из K8s с политикой в Vault.

---

5. Полезные "фишки" для отладки

- **Проверка прав:**  
    `vault token capabilities <token> secret/data/my-app` — Показывает, что конкретный токен может делать с этим путем (read, list, deny?). Очень помогает, когда приложение получает 403 ошибку.
- **Web UI:**  
    Если ты в Minikube, выполни:  
    `kubectl port-forward svc/vault 8200:8200 -n vault`  
    Теперь открой в браузере `http://localhost:8200`. Там можно делать всё то же самое, но мышкой.

Самый частый сценарий ошибки:

Если ты пишешь `vault kv get secret/my-app` и получаешь ошибку, проверь путь. В KV v2 Vault внутри себя добавляет слово **data**.

- Команда: `vault kv get secret/my-app`
- Путь в политике: `path "secret/data/my-app"` (это важно!).