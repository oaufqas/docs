**Master-нода (Control Plane)** — это мозг и управляющий центр кластера. Её главная задача — поддерживать желаемое состояние инфраструктуры (количество подов, сети, балансировку), реагировать на события и управлять Worker-нодами.

На каждой мастер-ноде в ручной сборке разворачиваются **4 ключевые службы (демоны)**, работающие как системные сервисы `systemd`:

### 1. kube-apiserver (Центральный шлюз)

- **Что это:** Единственный компонент Control Plane, который имеет прямой доступ к базе данных `etcd`. Является REST API-сервером.
- **Задачи:** Валидирует и настраивает данные для объектов (поды, сервисы, репликасеты), обрабатывает запросы от `kubectl` и воркеров.

`/etc/systemd/system/kube-apiserver.service`:

```ini
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=192.168.100.10 \
  --apiserver-count=1 \
  --allow-privileged=true \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/audit.log \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --client-ca-file=/var/lib/kubernetes/ca.pem \
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --etcd-cafile=/var/lib/kubernetes/ca.pem \
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \
  --etcd-servers=https://192.168.100.10:2379,https://192.168.100.12:2379 \
  --event-ttl=1h \
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \
  --runtime-config='api/all=true' \
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \
  --service-account-issuer=https://server.kubernetes.local:6443 \
  --service-node-port-range=30000-32767 \
  --service-cluster-ip-range=10.32.0.0/24 \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

1. `--authorization-mode=Node,RBAC` (Фильтр безопасности)
- **За что отвечает:** Включает двухэтапную проверку прав.
    - `Node` — специальный режим, который разрешает воркерам (`kubelet`) изменять только свои собственные параметры (чтобы взломанный Воркер-1 не мог удалить Воркер-2).
    - `RBAC` (Role-Based Access Control) — включает проверку прав на основе ролей. Именно он считывает поле `O: system:masters` в вашем сертификате админа и дает вам права корня.

2. `--bind-address=0.0.0.0`
- **За что отвечает:** Приказывает ядру открыть порт **6443** (главный порт Kubernetes) абсолютно на всех доступных сетевых интерфейсах мастера (и на `127.0.0.1`, и `192.168.100.10`). Благодаря этому вы можете управлять кластером со своего ноутбука по воздуху.

3. `--enable-admission-plugins=...,NodeRestriction,...` (Плагины допуска)
- **За что отвечает:** Admission-плагины — это судейская коллегия ядра Кубера. Когда вы просите создать под, запрос сначала проходит проверку RBAC (можно ли вам вообще создавать поды), а затем падает наAdmission-плагины. Они проверяют бизнес-логику: например, `LimitRanger` проверяет, не запросил ли под больше памяти, чем разрешено в этом namespace, а `NodeRestriction` жестко ограничивает права воркеров.

4. `--service-cluster-ip-range=10.32.0.0/24`
- **За что отвечает:** Это виртуальная подсеть, из которой Kubernetes будет выделять IP-адреса для внутренних абстракций — **Сервисов (Services)**. Эти адреса физически не существуют на сетевых картах, ядро Linux будет перехватывать их через `iptables` и перенаправлять на реальные поды. Первым адресом в этой сети автоматически станет `10.32.0.1` — внутренний IP самого API-сервера, который мы зашивали в его сертификат.

5. `--service-account-signing-key-file` и `--service-account-issuer`
- **За что отвечает:** Включают механизм генерации токенов безопасности для контейнеров (подов). API-сервер использует приватный ключ `service-account-key.pem` для криптографической подписи JWT-токенов, которые поды используют для подтверждения своей личности

6. `--etcd-servers`: Список URL-адресов всех нод etcd кластера для хранения состояния.

7. `--advertise-address`: IP-адрес конкретной мастер-ноды, который рекламируется воркерам.

8. `--client-ca-file` / `--tls-cert-file`: Пути к mTLS-сертификатам для шифрования всего входящего трафика.


### 2. etcd (Распределенное хранилище)

- **Что это:** Отказоустойчивая, распределенная база данных типа «ключ-значение». Хранит абсолютно все секреты, конфигурации и текущее состояние кластера.
- **Задачи:** Обеспечивает консистентность данных с помощью алгоритма консенсуса **Raft**.

`/etc/systemd/system/etcd.service`:

```ini
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name=control-plane-1-etcd \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls=https://192.168.100.12:2380 \
  --listen-peer-urls=https://192.168.100.12:2380 \
  --listen-client-urls=https://192.168.100.12:2379,https://127.0.0.1:2379 \
  --advertise-client-urls=https://192.168.100.12:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster=control-plane-0-etcd=https://192.168.100.10:2380,control-plane-1-etcd=https://192.168.100.12:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

1. `--listen-client-urls` (Что слушаем от программ)
- **За что отвечает:** Указывает IP-адреса и порты, на которых процесс `etcd` открывает сетевые сокеты для приема входящих запросов от клиентов (в нашем случае — от `kube-apiserver` и утилиты `etcdctl`).

2. `--advertise-client-urls` (Что советуем клиентам)
- **За что отвечает:** «Рекламный» адрес для клиентов. В распределенном кластере (когда мастеров 3 штуки), если клиент стучится на Мастер-1, этот `etcd` возвращает ему список адресов, говоря: если я умру, ищи мои копии по этим официальным адресам.

3. `--listen-peer-urls` (Что слушаем от других etcd)
- **За что отвечает:** Открывает сетевой сокет на порту **2380**. Этот порт используется **исключительно для связи между самими узлами `etcd`** (Peer-to-Peer) для синхронизации базы по алгоритму консенсуса Raft. Сюда клиенты (API-сервер) ломиться не имеют права.

4. `--initial-advertise-peer-urls` (Что заявляем другим etcd)
- **За что отвечает:** Адрес, который данный узел `etcd` отправляет своим соседям по кластеру.

5. `--initial-cluster`
- **За что отвечает:** Жесткая карта всей распределенной сети при первом старте базы.
- Если в кластере один мастер, то `control-plane=https://192.168.100.10:2380`. Если бы мастеров было три, строка бы выглядела так: `master1=https://...:2380,master2=https://...:2380...`.

6. `--initial-cluster-token`
- **За что отвечает:** Секретное кодовое слово (у нас `etcd-k8s-token`). Защищает от случайного зацикливания сетей. Если в вашей домашней Wi-Fi сети кто-то рядом тоже запустит свой `etcd`, их базы не попытаются автоматически объединиться в один кластер, так как токены (пароли кластеров) не совпадут.

7. `--initial-cluster-state=new`: Инициализация новой чистой базы при первом старте.


### 3. kube-controller-manager (Диспетчер-регулятор)

- **Что это:** Сборник непрерывных циклов регулирования (контроллеров).
- **Задачи:** Сравнивает реальное состояние кластера с желаемым. Если упал под, `Node Controller` и `ReplicaSet Controller` замечают это через API-сервер и дают команду создать новый под.

`/etc/systemd/system/kube-controller-manager.service`:

```ini
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --bind-address=0.0.0.0 \
  --cluster-cidr=10.200.0.0/16 \
  --cluster-name=kubernetes-the-hard-way \
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \
  --root-ca-file=/var/lib/kubernetes/ca.pem \
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --use-service-account-credentials=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

1. `--cluster-cidr=10.200.0.0/16`
- **За что отвечает:** **Самый главный сетевой флаг для подов.** Это гигантская виртуальная сеть, в которой будут жить ваши контейнеры. Контроллер-менеджер берет эту сеть `/16` (65 536 адресов) и автоматически нарезает её на маленькие кусочки `/24` (по 256 адресов) персонально для каждого воркера.
- _Вспоминаем наш файл `machines.txt`:_ Воркер-0 получил подсеть `10.200.0.0/24`, а Воркер-1 — `10.200.1.0/24`. Контроллер-менеджер следит, чтобы поды на разных нодах никогда не получили одинаковые IP-адреса.

2. `--leader-elect=true` (Выборы лидера)
- **За что отвечает:** Защита от конфликтов. Если вы запустите 3 мастера, на всех трех запустятся процессы `kube-controller-manager`. Чтобы они не начали одновременно отдавать противоположные приказы воркерам, этот флаг включает выборы. Бинарники делают микро-запись в `etcd` (берут блокировку). Тот, кто успел первым, становится Лидером (активным), а остальные два уходят в режим сна, готовые проснуться, если Лидер зависнет.

3. `--use-service-account-credentials=true`
- **За что отвечает:** Приказывает контроллерам внутри бинарника не использовать общие права root, а для каждой своей подзадачи (контроллер нод, контроллер подов) генерировать свой собственный изолированный Service Account с минимально необходимыми правами.

1. `--kubeconfig`: Путь к файлу авторизации (`kube-controller-manager.kubeconfig`), настроенному строго на локальный адрес `https://127.0.0.1:6443`.

### 4. kube-scheduler (Планировщик)

- **Что это:** Интеллектуальный алгоритм распределения нагрузки.
- **Задачи:** Отслеживает появление новых подов, у которых не назначена нода. Оценивает ресурсы воркеров (RAM, CPU, Affinity), фильтрует их и выбирает идеальный хост для запуска контейнера.

`/etc/systemd/system/kube-scheduler.service`

```ini
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \
  --leader-elect=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

- `--kubeconfig`: Путь к локальному кубконфигу для связи с API-сервером по адресу `https://127.0.0.1:6443`.
- `--leader-elect=true`: Флаг защиты от конфликтов планирования в Multi-Master топологии.

---

Все флаги из юнитов можно так же вынести в yaml конфиг файлы:

`/etc/kubernetes/config/kube-scheduler.yaml`:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
```

Тогда в юните нужно будет указать `--config=/etc/kubernetes/config/kube-scheduler.yaml`. Этот способ желательно использовать, если флагов при запуске много.

В итоге, мастер нода должна выглядеть так:

`/var/lib/kubernetes/kube-controller-manager.kubeconfig`
`/var/lib/kubernetes/kube-scheduler.kubeconfig`
`/etc/systemd/system/etcd.service`
`/etc/systemd/system/kube-apiserver.service`
`/etc/systemd/system/kube-controller-manager.service`
`/etc/systemd/system/kube-scheduler.service`

И [[notes/8_orchestration/kubernetes/installation|все нужные сертификаты]] в `/var/lib/kubernetes`

### Как компоненты общаются между собой

В архитектуре Kubernetes заложен принцип **строгой изоляции компонентов**. Ни один внутренний модуль не имеет права писать или читать из `etcd` напрямую, кроме `kube-apiserver`.

```
[ kubectl ] ──( HTTPS / Порт 6443 )──> [ kube-apiserver ] <───( mTLS )───> [ etcd ]
                                     ▲
 ┌───────────────────────────────────┼─────────────────────────────────────┐
 ▼                                   ▼                                     ▼
[ kube-scheduler ]       [ kube-controller-manager ]       [ kubelet на Воркерах ]
(mTLS / 127.0.0.1)             (mTLS / 127.0.0.1)           (через Балансировщик)
```

1. **Связующий узел:** Все компоненты (`scheduler`, `controller-manager`, `kubelet` на воркерах) общаются **исключительно с `kube-apiserver`**.
2. **Протокол обмена:** Взаимодействие идет по сетевому протоколу **HTTP/2 поверх TLS (mTLS)**. Каждый участник обязан предъявить свой персональный криптографический сертификат (паспорт), подписанный единым `ca.pem`.
3. **Локальная петля (Мастер):** Внутренние службы мастера (`scheduler` и `controller-manager`) обращаются к API-серверу по локальному адресу `https://127.0.0.1:6443` или через выделенный DNS-домен. Это гарантирует, что локальный трафик управления не покидает оперативную память сервера.
4. **Внешняя петля (Воркеры):** Агенты `kubelet` и `kube-proxy` с воркер-нод отправляют запросы на API-сервер через **внешний балансировщик трафика (HAProxy), если кластер содержит > 2 master node. Или на адрес единственной мастер ноды.**.

---

#### HA Multi-Master (настройки отказоустойчивости)

Переход от одиночной мастер-ноды к схеме с несколькими мастерами (High Availability Multi-Master) — это глубокая перестройка архитектуры Control Plane на 5 уровнях. 

1) При одной мастер ноде, на кластерный домен можно сразу вешать адрес единственного kube-apiserver. Но в HA режиме нужно отдельно поднимать балансировщик между всеми мастер нодами ([[notes/4_web-servers/nginx/basics|nginx]], [[notes/4_web-servers/haproxy/basics|haproxy]]).

```
               [ Внешний Балансировщик: server.k8s.local ]
                             (Порт 6443)
            ┌────────────────────┼────────────────────┐
            ▼                    ▼                    ▼
  [ control-plane-1 ]    [ control-plane-2 ]   [ control-plane-3 ]
(192.168.100.10:6443)   (192.168.100.20:6443)  (192.168.100.30:6443)
            │                    │                    │
            └─────────── mTLS HTTPS Кластер ──────────┘
             [ etcd-0:2379, etcd-1:2379, etcd-2:2379 ]

```

2) Перегенерация сертификата kube-apiserver'а. При генерации указываются все адреса, с которых будут поступать запросы, нужно дополнительно добавить туда адреса всех мастеров и адрес балансировщика.

```jinja2
-hostname=10.32.0.1,127.0.0.1,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster.local,server.kubernetes.local,192.168.100.9,{{ groups['masters'] | map('extract', hostvars, 'ansible_host') | join(',') }},{{ groups['masters'] | join(',') }}
```

3) Собрать все etcd в Raft-кластер, они должны начать постоянно общаться друг с другом, для этого нужно переделать конфигурацию. Сделать так, как показано в примере выше. Важно помнить что порт 2379 - для клиентов, 2380 - только для общения etcd между собой.
4) Компонентам управления необходимо передать флаг выбора лидерства `--leader-elect=true`. В кластере с одним мастером этот флаг должен быть `false`. Компоненты через API-сервер создадут специальный замок (Lease) в etcd. Двое мастеров будут спать в режиме ожидания (Standby), а работать будет только один. Если живой мастер умрет, двое оставшихся за миллисекунду выберут нового лидера и кластер продолжит работу без остановки. Это нужно сделать у `kube-controller-manager` и `kube-scheduler`. Внедрение этого флага в планировщик и контроллер-менеджер предотвращает шторм деплоев.
5) А сам `kube-apiserver` масштабируется без флагов — он не хранит состояние (Stateless), поэтому все 3 копии API-сервера могут работать одновременно. В их юнитах нужно только обновить флаг `--etcd-servers`, перечислив там через запятую IP-адреса всех трех нод etcd.


#### Проверка здоровья API-серверов и etcd

Самый главный диагностический инструмент, показывающий статус внутренних диспетчеров и базы данных:

```bash
kubectl get componentstatuses
```

_(Или сокращенно: `kubectl get cs`). Все строки (scheduler, controller-manager, etcd) должны иметь статус **Healthy**._


Проверка, что etcd на всех мастерах синхронизировался, выбрал лидера и работает без кворумных сбоев:

```bash
etcdctl \
  --cacert=ca.pem \
  --cert=kubernetes.pem \
  --key=kubernetes-key.pem \
  --endpoints=https://192.168.100.10:2379,https://192.168.100.12:2379 \
  endpoint health
```

_Каждая нода etcd в выводе терминала должна выдать строку: `... is healthy: successfully committed proposal`._

Опрос каждого сервера выборочно (минуя балансировщик):

```bash
# Опрашиваем первый мастер напрямую
kubectl get nodes --server=https://192.168.100.10:6443 --insecure-skip-tls-verify

# Опрашиваем второй мастер напрямую
kubectl get nodes --server=https://192.168.100.12:6443 --insecure-skip-tls-verify
```


#### Как рассчитывается формула кворума Raft

Это важнейшее условие, следуя из которого стоит выбирать ==нечетное== колличество мастеров. Чтобы etcd-кластер имел право принимать решения (запускать поды, читать ноды), в сети должно быть живо **строго большинство** участников. Формула минимального кворума выглядит так:

![[Pasted image 20260807031518.png]]

Где N — общее количество нод, изначально прописанных в кластере при старте флагом `--initial-cluster`.

Сценарий 1. У нас было 2 мастера

- **Изначально (N):** 2 ноды etcd.
- **Минимальный кворум (Q):** \[2/2] + 1 = 2.
- **Что произойдет:** 1 нода умерла. Живой осталась всего 1 нода. Так как 1 < 2, **кворум потерян**. Остаток кластера не имеет права принимать решения, так как он «не знает», жив ли его сосед или сеть просто разделилась (проблема Split-Brain). База уходит в таймаут.

Сценарий 2. Если у нас было 3 мастера (Продакшн-стандарт)

- **Изначально (N):** 3 ноды etcd.
- **Минимальный кворум (Q):** \[3/2] + 1 = 2.
- **Что произойдет:** 1 нода умирает. Живыми остаются 2 ноды из 3. Так как 2 ≥ 2, **кворум сохранен**! Две выжившие ноды перевыберут лидера за миллисекунды, и ваш кластер продолжит работать вообще без задержек.

> **Важное правило:** Кластер из 2 нод выдерживает **0 падений**. Кластер из 3 нод выдерживает **1 падение**. Кластер из 5 нод выдерживает **2 падения**. Именно поэтому четное количество мастеров в High Availability схемах никогда не используется.

Поэтому две мастер ноды в кластере не имеют смысла. Имеют смысл нечетные колличества (3, 5, 7)