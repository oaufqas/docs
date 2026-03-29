**Ingress-контроллер** — это специализированный балансировщик нагрузки и прокси-сервер, который управляет **внешним доступом** к сервисам внутри кластера Kubernetes. 

### Как это работает

В Kubernetes есть два понятия, которые часто путают:

1. **Ingress (Ресурс):** Это набор правил (манифест YAML), где вы описываете, какой входящий трафик на какой сервис (Service) нужно отправить (например: `://example.com` → `api-service`).
2. **Ingress-контроллер:** Это само приложение (программное обеспечение), которое читает эти правила и реально направляет трафик. Без контроллера правила Ingress просто не будут работать. 

Основные функции

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

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    # Указываем, какой контроллер должен обрабатывать этот Ingress
    kubernetes.io/ingress.class: nginx 
spec:
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