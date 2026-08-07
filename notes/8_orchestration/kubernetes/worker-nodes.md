**Worker-нода (Рабочий узел)** — это вычислительная мощность кластера. Её главная задача — физически запускать контейнеры приложений в изолированных пространствах имен, управлять их жизненным циклом, обеспечивать локальную сеть и выполнять команды, поступающие от Master-ноды (Control Plane).

На каждой Worker-ноде разворачивается слой контейнеризации (container-runtime) и два системных компонента Kubernetes, работающих под управлением `systemd`.

```
	   [ СИГНАЛЫ ОТ MASTER-НОДЫ (API-Server) ]
						 │
						 ▼
					[ kubelet ] 
						 │ (Интерфейс CRI)
						 ▼
				[ containerd ] <───(Интерфейс CNI)───> [ Calico / CNI Plugins ]
						 │
						 ▼
			[ runc (Кольцо защиты) ]
						 │
						 ▼
			[ Поды / Контейнеры приложений ]

```

### 1) kubelet (Главный надсмотрщик ноды)

- **Что это:** Основной агент Kubernetes, работающий на каждом воркере.
- **Задачи:** Принимает от API-сервера спецификации подов (`PodSpecs`). Дает команду рантайму запустить контейнеры, непрерывно следит за их здоровьем (Liveness/Readiness probes) и рапортует мастеру о статусе ноды.

`/etc/systemd/system/kubelet.service`:

```ini
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`/var/lib/kubelet/kubelet-config.yaml`:

```yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: "0.0.0.0"
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubelet/ca.pem"
authorization:
  mode: Webhook
cgroupDriver: systemd
containerRuntimeEndpoint: "unix:///var/run/containerd/containerd.sock"
enableServer: true
failSwapOn: false
maxPods: 32
memorySwap:
  swapBehavior: NoSwap
port: 10250
resolvConf: "/run/systemd/resolve/resolv.conf"
registerNode: true
runtimeRequestTimeout: "15m"
podCIDR: "10.200.0.0/24"
clusterDNS:
  - "10.32.0.10"
clusterDomain: "kubernetes.local"
podInfraContainerImage: "registry.k8s.io/pause:3.9"
tlsCertFile: "/var/lib/kubelet/worker-0.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/worker-0-key.pem"
```

- `podCIDR`: Уникальная подсеть подов, выделенная строго для данного воркера (например, `10.200.0.0/24` для `node-0`).
- `clusterDomain`: Официальный внутренний суффикс кластера (`cluster.local`).
- `clusterDNS`: IP-адрес службы CoreDNS (`10.32.0.10`).
- `containerRuntimeEndpoint`: Путь к Unix-сокету рантайма (для containerd: `unix:///var/run/containerd/containerd.sock`).

### 2) kube-proxy (Сетевой диспетчер)

- **Что это:** Сетевой регулировщик воркера, отражающий концепцию "Сервисов" Kubernetes.
- **Задачи:** Следит за появлением объектов `Service` и `Endpoints` на Мастере. На основе этих данных нарезает правила внутри ядра хоста, чтобы трафик на виртуальный IP (например, `ClusterIP`) балансировался и долетал до реальных подов.
- **Режимы работы:** `iptables` (нарезка последовательных правил цепочек) или `ipvs` (использование встроенного в Linux L4-балансировщика для высоконагруженных кластеров).

`/etc/systemd/system/kube-proxy.service`:

```ini
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --kubeconfig=/var/lib/kube-proxy/kubeconfig \
  --proxy-mode=iptables \
  --cluster-cidr=10.200.0.0/16
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

1. **`--config=/var/lib/kube-proxy/kube-proxy-config.yaml`**  
Указывает путь к главному YAML-файлу настроек. Если этот флаг указан, большинство остальных флагов переносятся внутрь YAML.
2. **`--kubeconfig=/var/lib/kube-proxy/kube-proxy.kubeconfig`**  
Путь к сетевому паспорту авторизации, который мы генерировали силами Ansible (внутри зашиты ключи `system:kube-proxy` и внешний URL балансировщика мастера).
3. **`--proxy-mode=iptables`** (или `ipvs`)  
**Самый критичный флаг.** Задает режим работы сетевого движка:
- `iptables` (стандарт): `kube-proxy` непрерывно пишет правила фильтрации в брандмауэр ядра Linux хоста.
- `ipvs`: Использует встроенный в ядро Linux балансировщик Netfilter L4. Идеален, если в кластере планируются тысячи сервисов, так как работает быстрее, чем миллион строк в `iptables`.
1. **`--cluster-cidr=10.200.0.0/16`**  
Глобальная подсеть всех ваших подов. `kube-proxy` обязан знать этот пул, чтобы отличать внутрикластерный трафик от внешнего интернета и правильно делать маскарадинг (SNAT).
2. **`--hostname-override=node-0`**  
Принудительно переопределяет имя ноды в системе. Должно строго совпадать с именем хоста в инвентаре Ansible и флагом `--hostname-override` в `kubelet`, иначе Мастер не сможет связать сетевые правила с конкретным сервером.
3. **`--bind-address=0.0.0.0`**  
На каком IP-адресе хоста утилита открывает свои служебные порты (например, порт метрик `10249` или порт проверки здоровья `10256`). `0.0.0.0` означает слушать все интерфейсы.
4. **`--v=2`**  
Уровень детализации логов, которые улетают в `journald`. Лог уровня `2` оптимален для продакшна: показывает изменения в сервисах, но не забивает диск.


### 3) containerd (Слой контейнеризации / Runtime)

- **Что это:** Легковесный промышленный рантайм, который управляет полным жизненным циклом контейнеров на хосте.
- **Задачи:** Скачивает образы с реестров (Docker Hub и др.), управляет хранилищем дисков контейнеров, передает сетевые команды в CNI и вызывает низкоуровневый движок `runc`.
- **Взаимодействие (CRI):** Общается с `kubelet` по протоколу **CRI** (Container Runtime Interface) через gRPC-запросы.

`/etc/systemd/system/containerd.service`:

```ini
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
```

`/etc/containerd/config.toml`:

```toml
version = 2

[plugins."io.containerd.grpc.v1.cri"]
  [plugins."io.containerd.grpc.v1.cri".containerd]
    snapshotter = "overlayfs"
    default_runtime_name = "runc"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
[plugins."io.containerd.grpc.v1.cri".cni]
  bin_dir = "/opt/cni/bin"
  conf_dir = "/etc/cni/net.d"
```

### 4) runc (Исполнитель)

- **Что это:** Низкоуровневая утилита (CLI), созданная по стандарту OCI (Open Container Initiative).
- **Задачи:** Напрямую взаимодействует с ядром Linux хоста. Именно `runc` создает `Namespaces` (изоляцию сети, процессов, пользователей) и `cgroups` (ограничения по CPU/RAM), после чего запускает процесс вашего приложения, обеспечивая его работу в пользовательском пространстве.

В итоге на worker нодах должны быть:

`/var/lib/kubelet/kubeconfig`
`/var/lib/kubelet/kubelet-config.yaml`
`/var/lib/kube-proxy/kubeconfig`
`/etc/containerd/config.toml`
`/etc/systemd/system/containterd.service`
`/etc/systemd/system/kubelet.service`
`/etc/systemd/system/kube-proxy.service`

И [[notes/8_orchestration/kubernetes/installation|все нужные сертификаты]] в `/var/lib/kubelet`

---

#### Как Worker-нода работает с сетью (Эволюция CNI)

Сетевая архитектура воркера полностью делегирована стандарту **CNI** (Container Network Interface).

1. **Локальные утилиты (`/opt/cni/bin/`):** Голые бинарники (`bridge`, `loopback`, `portmap`). Они умеют только локально нарезать виртуальные интерфейсы в рамках одного Linux-хоста.
2. **Высокоуровневый CNI-провайдер (Calico):** Запускается как `DaemonSet` (системные поды на каждом воркере).
    - **IPAM:** Читает флаг `podCIDR` из `kubelet` и автоматически нарезает IP-адреса для новых подов.
    - **veth-пары:** Создает виртуальный кабель. Один конец (`eth0`) помещает внутрь сетевого пространства пода, второй (`cali...`) оставляет снаружи.
    - **VXLAN / Overlay:** Упаковывает межхостовый трафик подов в стандартные UDP/IP пакеты (порт 4789). Это позволяет подам с `node-0` общаться с подам на `node-1` через ваш домашний Wi-Fi роутер, который даже не подозревает о существовании подсети `10.200.0.0/16`.


#### Настройки конфигурации и предтребования к ОС

Чтобы компоненты воркера работали стабильно и не вызывали паники ядра Linux, операционная система хоста должна быть подготовлена по строгому стандарту:

- **Абсолютное отключение Swap (`swapoff -a`):** Планировщик Kubernetes (`kube-scheduler`) рассчитывает CPU и RAM с аптекарской точностью. Если воркер начнет сбрасывать память подов на медленный диск в swap-раздел, нарушится вся логика лимитов (`Limits/Requests`), а нода начнет дико тормозить.
- **Включение маршрутизации ядра (`sysctl -w net.ipv4.ip_forward=1`):** По умолчанию Linux запрещает пересылать пакеты между разными сетевыми картами. Для CNI этот флаг обязателен, чтобы ядро хоста разрешало пересылать пакеты из сетевой карты `eth0` внутрь виртуальных интерфейсов подов.
- **Изоляция доменов (`clusterDomain: "cluster.local"`):** Kubelet принудительно генерирует для каждого нового контейнера файл `/etc/resolv.conf`, зашивая туда IP-адрес CoreDNS (`10.32.0.10`) и поисковые суффиксы. Благодаря этому поды получают способность мгновенно видеть соседние сервисы по коротким именам.

#### Что конкретно произойдет, если оставить swap включенным?

1. Сломается планировщик кластера (`kube-scheduler`)

Планировщик распределяет поды по воркерам на основе жестких математических расчетов свободной оперативной памяти (параметры `Requests` и `Limits`). Он верит, что на ноде есть, например, 2 ГБ _быстрой_ физической RAM.  
Если память кончится и ядро Linux начнет тихо сбрасывать данные подов на медленный SSD/HDD в swap, для Kubernetes эта память по-прежнему будет считаться «свободной». Но поды внутри кластера начнут дико тормозить, ловить таймауты и зависать, а планировщик даже не поймет, почему так происходит.

2. Контейнеры начнут хаотично умирать

В Linux за переполнение памяти отвечает киллер ядра — **OOM Killer** (_Out of Memory Killer_).

- Без swap: когда под превышает свой `Memory Limit`, Kubernetes мгновенно и чисто убивает конкретный контейнер с понятной ошибкой `OOMKilled`.
- Со swap: ядро Linux до последнего пытается спасти систему и размазывает память подов по swap-файлу. В итоге вместо изоляции конкретного нарушителя начинает тормозить **вся операционная система воркера целиком**, включая сам `kubelet` и `containerd`. Нода может полностью перестать отвечать на запросы мастеров.

Проверить включен ли swap легко:

```bash
free -m
# swap должен быть по нулям
swapon --show
# команда не должна выводить ничего на экран
```

Выключить его можно:

```bash
swapoff -a

sudo nano /etc/fstab # -> закомментировать строку со swap, это отключит его из автозагрузки ядра
```