# StatefulSet

#### 4 ключевые особенности StatefulSet

1. **Стабильное имя (Network ID):** Поды именуются по порядку: `web-0`, `web-1`, `web-2`. Если под `web-0` упадет, новый под получит то же самое имя.
2. **Стабильное хранилище (Storage):** Каждый под получает свой собственный том (PersistentVolume). Если под переезжает на другой узел, его диск «переезжает» вместе с ним.
3. **Порядок запуска и удаления:** Поды запускаются строго по очереди (0, затем 1, затем 2) и удаляются в обратном порядке. Это критично для кластеров (например, чтобы сначала запустить Master, а потом Slaves).
4. **Headless Service:** Для работы StatefulSet требуется специальный сервис без IP-адреса, который позволяет обращаться к конкретному поду по DNS-имени (например, `web-0.nginx-service`).

Пример манифеста:

```yaml
# 1. Headless Service для сетевой идентификации
apiVersion: v1
kind: Service
metadata:
  name: nginx-hs
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None # Ключевая деталь: делает сервис Headless
  selector:
    app: nginx
---
# 2. Сам StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx-hs" # Привязка к Headless сервису
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:stable-alpine
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  # Автоматическое создание дисков для каждого пода
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

Пример разыертывания MySQL в StatefulSet:

```yaml
# Headless Service для стабильного сетевого имени
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
  clusterIP: None # Делаем сервис "безголовым"
  selector:
    app: mysql

---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: "mysql"
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: secret-variables
              key: MYSQL_ROOT_PASSWORD
              
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: secret-variables
              key: DB_NAME

        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: secret-variables
              key: DB_USER

        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: secret-variables
              key: DB_PASSWORD

        ports:
        - containerPort: 3306
          name: mysql
          
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql # Куда MySQL пишет данные
          
        - name: config-mysql
          mountPath: /etc/mysql/conf.d/
          
        - name: init-scripts
          mountPath: /docker-entrypoint-initdb.d/
          
      volumes:
        - name: init-scripts
          configMap:
            name: initsql
        - name: config-mysql
          configMap:
            name: config-mysql
          
  # Автоматическое создание диска
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

1. **Сетевой адрес**: После запуска этот под будет доступен внутри кластера по адресу `mysql-0.mysql-hs`. Это имя **не изменится**, даже если под переедет на другой узел.
2. **`volumeClaimTemplates`**: Minikube автоматически создаст виртуальный диск (PVC) размером 1ГБ. Даже если вы удалите StatefulSet командой `kubectl delete`, этот диск **останется**, и ваши данные в `/var/lib/mysql` не пропадут.
3. **Переменные окружения**: Мы берем пароль из `Secret`, что безопаснее, чем писать его открытым текстом в манифесте.
4. В `StatefulSet` поле `serviceName` должно **строго совпадать** с `metadata.name` вашего Headless-сервиса.

# DaemonSet 

##### Это особый контроллер в Kubernetes, чья единственная задача — гарантировать, что **на каждом узле (Node)** кластера запущена ровно одна копия (Pod) определенного приложения.

Если в кластер добавляется новый сервер (нода) — DaemonSet тут же автоматически запускает на нем свой Под. Если сервер удаляется — Под исчезает вместе с ним.

---

Для чего это нужно? (Типичные примеры)

Обычно это «служебные» задачи, которые должны работать везде:

1. **Сбор логов:** Например, `Fluentd` или `Logstash`. Они должны сидеть на каждой ноде, чтобы собирать логи всех остальных контейнеров на этой машине.
2. **Мониторинг:** Например, `Prometheus Node Exporter`. Он собирает метрики железа (CPU, RAM) конкретного сервера.
3. **Сетевые плагины:** Например, `Calico` или `Flannel`. Они настраивают сеть на каждой ноде, чтобы Поды могли общаться друг с другом.
4. **Хранилища (Storage):** Драйверы для работы с дисками (например, `Ceph` или `GlusterFS`).

---

Главные отличия от Deployment

| Особенность          | Deployment                                                 | DaemonSet                                                               |
| -------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------- |
| **Количество копий** | Ты задаешь сам (`replicas: 3`)                             | Равно количеству узлов (автоматически)                                  |
| **Где запускать?**   | Планировщик (Scheduler) выбирает наименее загруженные узлы | Запускается на **каждом** узле (игнорируя обычные правила планировщика) |
| **Масштабирование**  | Вручную или через HPA                                      | При изменении размера кластера                                          |

---

Тонкие настройки (Taints и Selectors)

Иногда нам не нужно запускать DaemonSet **вообще на всех** узлах. Для этого есть два механизма:

- **Node Selector:** Можно сказать: «Запускайся только на узлах с меткой `ssd=true`».
- **Tolerations:** По умолчанию на Master-ноды (где живет «мозг» Кубера) обычные Поды не пускают (там стоит Taint). DaemonSet умеет «игнорировать» этот запрет, чтобы мониторить даже системные узлы.

---

Пример в манифесте

Выглядит почти как Deployment, но без поля `replicas`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        # Тут обычно монтируют папки с логами самой Ноды
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```


Обновление (Rolling Update)

Когда ты меняешь образ в DaemonSet, он не убивает все Поды разом. Он удаляет старый Под на одной ноде, создает новый, ждет, пока тот станет `Ready`, и только потом переходит к следующей ноде. Это позволяет обновлять системные утилиты без простоя.

---

# Jobs (Разовая задача)

**Job** создает один или несколько Подов и гарантирует, что определенное количество из них успешно завершится (код выхода 0).

Когда использовать?

- Миграции базы данных при деплое.
- Генерация тяжелого отчета или PDF.
- Пакетная обработка данных (Batch processing).
- Загрузка бэкапа в облако.

Ключевые параметры Job:

1. **`backoffLimit`**: Сколько раз K8s будет пытаться перезапустить Под, если тот упал с ошибкой. По умолчанию — 6 раз.
2. **`completions`**: Сколько успешных завершений нужно, чтобы Job считалась выполненной. Например, если нужно обработать 10 очередей.
3. **`parallelism`**: Сколько Подов могут работать одновременно. Если `completions: 10`, а `parallelism: 2`, Job запустит 2 пода, и как только один закончит, запустит следующий, пока не наберется 10 успешных.
4. **`activeDeadlineSeconds`**: Лимит времени. Если задача зависла и не успела за X секунд — K8s её принудительно убивает.

Пример манифеста:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: my-app:latest
        command: ["npm", "run", "migrate"]
      restartPolicy: OnFailure # Важно: Never или OnFailure (но не Always!)
  backoffLimit: 4
```

# CroneJobs (Задачи по расписанию)

**CronJob** — это надстройка над Job. Она просто запускает обычную Job по расписанию, используя стандартный синтаксис **crontab** (пять звездочек).

Когда использовать?

- Ежедневный бэкап в 2 часа ночи.
- Рассылка писем-напоминаний по пятницам.
- Очистка кэша или временных файлов раз в час.

Формат расписания (Schedule):

`"минута час день месяц день_недели"`

- `"0 0 * * *"` — каждый день в полночь.
- `"*/5 * * * *"` — каждые 5 минут.

Важные настройки CronJob:

1. **`concurrencyPolicy`**: Что делать, если пришло время нового запуска, а старая Job еще не закончила?
    - `Allow` (по умолчанию): запустить еще одну параллельно.
    - `Forbid`: пропустить новый запуск, пока старый не доделает.
    - `Replace`: убить старую Job и запустить новую.
2. **`startingDeadlineSeconds`**: Если кластер был выключен или перегружен, и мы пропустили время запуска — до какого момента попытка еще считается актуальной?
3. **`successfulJobsHistoryLimit`** и **`failedJobsHistoryLimit`**: Сколько завершенных объектов Job хранить в истории (чтобы можно было посмотреть логи). Обычно ставят 3 и 1.
4. Если за время downtime кластера, количество job, запущенных CronJob'ой будет > 100, крон жоба выдаст ошибку и не будет больше пытаться запускать жобы. 

Пример манифеста:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *" # Каждый день в 02:00
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup-tool
            image: backup:1.0
          restartPolicy: OnFailure
  concurrencyPolicy: Forbid # Не запускать второй, если первый тормозит
```

---

#### **Autoscaling (HPA - HorizontalPodAutoscaler)** — автоматическое поднятие реплик, в зависимости от нагрузки на сервер

```yml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-autoscaling
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

###### Искуственная нагрузка на процессор

```bash
# Выбираем один из подов нашего деплоймента и грузим его
kubectl exec -it $(kubectl get pod -l project=rv -o name | head -n 1) -c f1 -- sh -c "while true; do md5sum /dev/urandom; done"
```

###### Мониторинг hpa

```bash
kubectl get hpa my-autoscaling --watch
```


---

### Probes - проверки жизни контейнеров внтури подов

- Пробы делает kubelet-агент на каждой ноде

#### Есть 3 вида проб:

- **LivenessProbe** - Постоянно выполняется и следит за жизнью подов
- **ReadinessProbe** - Проверяет, готов ли под к работе, пока не будет готово, трафик на этот под идти не будет
- **StartupProbe** - Пока эта проба не пройдет, остальные (Liveness и Readiness) **отключены**.

Примеры:

```yaml
spec: 
  containers:
  - name: Ubuntu
    image: ubuntu
    args:
    - /bin/bash
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600;
    livenessProbe: # Постоянная проверка жив ли контейнер
    readinessProbe: # Проверка когда можно отправлять трафик на контейнер
      exec: # Exec проверка
        command: # Command == ENTRYPOINT, args == CMD
        - cat 
        - /tmp/healthy
      tcpSocket: # TCP проверка
        port: 8001
      httpGet: # HTTP GET проверка
        path: /healthcheck
        port: 8000
          
    initialDelaySeconds: 5 # Колличество секунд от старта до пробы
    periodSeconds: 5 # Длительность времени между двумя проведениями проб
    timeoutSeconds: 1 # Колличество секунд ожидания пробы
    successThreshold: 1 # Минимальное колличество последовательных проверок
    failureThreshold: 3 # Максимальное колличество перезапусков при ошибках
```

```yaml
spec:
  template:
    spec:
      containers:
        - name: {{ .Values.server.name }}
          image: "{{ .Values.server.image.repository }}:{{ .Values.server.image.tag }}"
          ports:
            - containerPort: {{ .Values.server.port }}
          
          # 1. Startup Probe: Даем серверу время "проснуться"
          startupProbe:
            httpGet:
              path: /
              port: {{ .Values.server.port }}
            failureThreshold: 30 # Даем 30 попыток
            periodSeconds: 10    # Раз в 10 секунд (итого 5 минут на старт)

          # 2. Readiness Probe: Проверяем доступность базы через роут /
          readinessProbe:
            httpGet:
              path: /
              port: {{ .Values.server.port }}
            initialDelaySeconds: 5 # Начать проверку через 5 сек после Startup
            periodSeconds: 5      # Проверять каждые 5 сек
            failureThreshold: 3   # Если 3 раза упало — убрать из трафика

          # 3. Liveness Probe: Проверяем, что сервер вообще не завис
          livenessProbe:
            httpGet:
              path: /
              port: {{ .Values.server.port }}
            initialDelaySeconds: 15
            periodSeconds: 20
```