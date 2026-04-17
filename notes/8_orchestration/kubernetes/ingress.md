**Ingress-контроллер** — это специализированный балансировщик нагрузки и прокси-сервер, который управляет **внешним доступом** к сервисам внутри кластера Kubernetes. 

### Как это работает

В Kubernetes есть два понятия, которые часто путают:

1. **Ingress (Ресурс):** Это набор правил (манифест YAML), где вы описываете, какой входящий трафик на какой сервис (Service) нужно отправить (например: `://example.com` → `api-service`).
2. **Ingress-контроллер:** Это само приложение (программное обеспечение), которое читает эти правила и реально направляет трафик. Без контроллера правила Ingress просто не будут работать. 

Основные функции

- По сути ingress контроллер это высокоуровневая надстройка над конфигами web-серверов, например nginx.conf, все описывается как сущность ingress в манифесте.
- **Маршрутизация по путям и доменам:** Позволяет использовать один IP-адрес для десятков разных сервисов (L7-балансировка).
- **Терминация SSL/TLS:** Контроллер берет на себя расшифровку трафика, избавляя от этой нагрузки сами приложения.
- **Балансировка нагрузки:** Распределяет запросы между репликами ваших приложений.
- **Безопасность:** Позволяет настраивать аутентификацию и ограничения по IP.

![[Pasted image 20260327115844.png|697]]

---

## Популярные решения

В отличие от многих других компонентов K8s, контроллер не устанавливается автоматически. Вам нужно выбрать и установить его самостоятельно: 

##### 1. **NGINX Ingress Controller:** Самый популярный стандарт, но стоит учитывать, что проект сообщества `ingress-nginx` в начале 2026 года перешел в режим ограниченной поддержки.

- **Добавьте репозиторий:**

```bash
helm repo add ingress-nginx https://kubernetes.github.io
helm repo update
```

- **Установите контроллер:**

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
--namespace ingress-nginx --create-namespace
```

- **Проверка:** `kubectl get pods -n ingress-nginx`.

---

##### 2. **Traefik:** Современный контроллер с автоматическим обновлением конфигурации и поддержкой Docker/K8s из коробки.

- **Добавьте репозиторий:**

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

- **Установите контроллер:**

```bash
helm install traefik traefik/traefik \
--namespace traefik --create-namespace \
--set dashboard.enabled=true
```

---

##### 3. **HAProxy:** Высокопроизводительное решение для больших нагрузок.

- **Добавьте репозиторий:**

```bash
helm repo add haproxy-ingress https://haproxy-ingress.github.io/charts
helm repo update
```

- **Установите контроллер:**

```bash
helm install haproxy-ingress haproxy-ingress/haproxy-ingress \
--namespace ingress-controller --create-namespace
```

---

##### 4. **Istio Ingress Gateway:** Часть Service Mesh для сложной маршрутизации и безопасности.
##### 5. **Облачные решения:** У провайдеров (AWS, Google Cloud, Yandex Cloud) есть свои контроллеры, интегрированные с их облачными балансировщиками.

---

#### После того как контроллер запущен, вам нужно создать объект **Ingress**, чтобы направить трафик на ваше приложение:

Пример 1

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx # Устарело
spec:
    # Указываем, какой контроллер должен обрабатывать этот Ingress
  ingressClassName: nginx
  rules:
  - host: my-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

Пример 2

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    # Полезная аннотация для перенаправления на SSL (если есть сертификат)
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nignx
  rules:
  - host: myapp.com             # 1. Проверка домена
    http:
      paths:
      - path: /api              # 2. Проверка пути (префикс)
        pathType: Prefix
        backend:
          service:
            name: api-service   # 3. Куда отправить
            port:
              number: 8080      # 4. На какой порт Сервиса
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-frontend
            port:
              number: 80
```

За что отвечают поля

1. **`annotations`**: Это «инструкции» для контроллера. Например, через них настраиваются лимиты (Rate Limit), таймауты, базовое шифрование (Basic Auth) или работа с сертификатами Let's Encrypt.
2. **`host`**: Ingress смотрит на HTTP-заголовок `Host` в запросе. Если пришел запрос на `other-app.com`, а в правилах только `myapp.com`, контроллер выдаст 404.
3. **`pathType`**:
    - `Prefix`: Все, что начинается с этого пути (например, `/api/users`, `/api/v2`).
    - `Exact`: Строгое соответствие (только `/api`).
    - `ImplementationSpecific`: Зависит от настроек конкретного контроллера.
4. **`backend`**: Конечная точка. Здесь мы указываем имя **Service** (не Пода!) и его порт.

---

Как Ingress понимает, на какой порт отправлять запрос? Логика такая:

1. **Вход в контроллер:** Пользователь всегда стучится на 80 (HTTP) или 443 (HTTPS) порты самого Ingress Controller.
2. **Поиск порта Service:** В блоке `backend.service.port.number` вы указываете порт, который прописан в поле `port` вашего **Service**.
3. **TargetPort:** Service перенаправляет запрос на `targetPort` (порт самого приложения внутри контейнера).

**Пример связки:**

- Приложение слушает внутри контейнера порт **3000**.
- Service имеет `port: 80` и `targetPort: 3000`.
- Ingress имеет `port: { number: 80 }`.
- **Итог:** Пользователь заходит на `myapp.com` (порт 80 по умолчанию), Ingress кидает запрос на Service порт 80, тот перекидывает на контейнер порт 3000.

---

TLS (HTTPS)

Чтобы работал замок в браузере, в Ingress добавляется секция `tls`:

```yaml
spec:
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-tls-secret # Имя Secret, где лежат сертификат и ключ
```


---

### SSL certificates let's Encrypt

Установка cert-manager c плагином Yandex Cloud DNS ACME webhook

```bash
helm pull oci://cr.yandex/yc-marketplace/yandex-cloud/cert-manager-webhook-yandex/cert-manager-webhook-yandex \ 
--version 1.0.9 \ 
--untar && \ 
helm install \ 
--namespace <пространство_имен> \ 
--create-namespace \ 
--set-file config.auth.json=key.json \ 
--set config.email='<адрес_электронной_почты_для_уведомлений_от_Lets_Encrypt>' \ 
--set config.folder_id='<идентификатор_каталога_с_зоной_Cloud_DNS>' \ 
--set config.server='URL_сервера_Lets_Encrypt' \ 
# https://acme-staging-v02.api.letsencrypt.org/directory Тестовый URL
# https://acme-v02.api.letsencrypt.org/directory Основной URL
cert-manager-webhook-yandex ./cert-manager-webhook-yandex/
```

Получение тестового сертификата:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: domain-name
  namespace: <пространство_имен>
spec:
  secretName: domain-name-secret
  issuerRef:
    # ClusterIssuer, созданный вместе с Yandex Cloud DNS ACME webhook.
    name: yc-clusterissuer
    kind: ClusterIssuer
  dnsNames:
    # Домен должен входить в вашу публичную зону Cloud DNS.
    # Указывается имя домена (например, test.example.com), а не имя DNS-записи.
    - <имя_домена>
```

Результат должен быть:

```bash
kubectl get certificate


NAME         READY  SECRET              AGE
domain-name  True   domain-name-secret  45m
```

Получение сертификата:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: http01-clusterissuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <электронная_почта_для_уведомлений_от_Lets_Encrypt>
    privateKeySecretRef:
      name: http01-clusterissuer-secret
    solvers:
    - http01:
        ingress:
          class: nginx
```

Объекты для проверки cert-manager:

```yaml
apiVersion: networking.k8s.io/v1 
kind: Ingress 
metadata: 
  name: minimal-ingress 
  annotations: cert-manager.io/cluster-issuer: "http01-clusterissuer" 
spec: 
  ingressClassName: nginx 
  tls: 
    - hosts: 
      - <URL_адрес_вашего_домена> 
      secretName: domain-name-secret 
  rules: 
    - host: <URL_адрес_вашего_домена> 
      http:
        paths: 
        - path: / 
          pathType: Prefix 
          backend: 
            service: 
              name: app 
              port: number: 80 
   
--- 

apiVersion: v1 
kind: Service 
metadata: 
  name: app 
spec: 
  selector: 
    app: app 
  ports: 
  - protocol: TCP 
    port: 80 
    targetPort: 80 
    
---

apiVersion: apps/v1 
kind: Deployment 
metadata: 
  name: app-deployment 
  labels: 
    app: app 
spec: 
  replicas: 1 
  selector: 
    matchLabels: 
      app: app 
  template: 
    metadata: 
      labels: 
      app: app 
    spec: 
      containers: 
        - name: app 
          image: nginx:latest 
          ports:
            - containerPort: 80

```


---

Полезные команды

- `kubectl get ingress` — посмотреть внешние адреса.
- `kubectl describe ingress <name>` — если Ingress не работает, тут будет видно, нашел ли он нужный Service.
- `kubectl get ingressclass` — посмотреть ingress controllers