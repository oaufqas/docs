![[Pasted image 20260406172732.png]]

### [[managed-k8s|Installation k8s in yandex cloud]]


#### Install kubectl

1. Download latest release (Linux x86-64)

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

2. Install kubectl

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

3. test to installed

```bash
kubectl version --client
```


---

#### Kubernetes the hard way

- Настройка инфры
- Настройка виртуалок
- Создание mTLS сертификатов
- Создание kubeconfig
- Ключ шифрования данных в etcd
- Etcd
- Настройка control-plane
- Настройка worker-nodes
- Ручная настройка сети подов

Для ручной установки изначально необходимо настроить минимум 3 сервера (или виртуальные машины), которые будут находиться в одной локальной сети и видеть друг друга. В моем примере виртуалки накатаны в домашнем [[notes/10_virtualization/proxmox/basics|proxmox]], и есть LXC контейнер для администрирования виртуалок: 

```
LXC Container (kubectl, ansible, terraform) - 192.168.100.111/24
control-plane (etcd, apiserver, sheduler, controller-manager) - 192.168.100.10/24
node-0 (kubelet, kubeproxy, containerd) - 192.168.100.11/24
node-1 (kubelet, kubeproxy, containerd) - 192.168.100.12/24
Proxmox server - 192.168.100.1 (default gateway), 192.168.0.167.
```

После создания виртуалок и настройки сети (у нод желательно должны быть статические ip и обязательно разные имена хостов), подключаемся к админской виртуалке, на которую изначально скачаем все бинарники, сгенерируем сертификаты и создадим конфиги.

Для удобства можно скопировать [репозиторий с шаблонами файлов и бинарниками](https://github.com/kelseyhightower/kubernetes-the-hard-way). 

```bash
wget -q --show-progress --https-only --timestamping -P downloads \
-i downloads-$(dpkg --print-architecture).txt
```

Распаковка архивов и раскладывание бинарников по категориям

```bash
{
cd downloads
ARCH=$(dpkg --print-architecture)
mkdir -p ./{client,cni-plugins,controller,worker}
tar -xvf crictl-v1.32.0-linux-${ARCH}.tar.gz -C ./worker/
tar -xvf containerd-2.1.0-beta.0-linux-${ARCH}.tar.gz --strip-components 1 -C ./worker/
tar -xvf cni-plugins-linux-${ARCH}-v1.6.2.tgz -C ./cni-plugins/
tar -xvf etcd-v3.6.0-rc.3-linux-${ARCH}.tar.gz -C ./ --strip-component 1 etcd-v3.6.0-rc.3-linux-${ARCH}/etcdctl etcd-v3.6.0-rc.3-linux-${ARCH}/etcd
mv ./{etcdctl,kubectl} ./client/
mv ./{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} ./controller/
mv ./{kubelet,kube-proxy} ./worker
mv ./runc.${ARCH} ./worker/runc
chmod +x ./{client,cni-plugins,controller,worker}/*
}
```

Создаем файл с перечислением созданой инфры и закидываем на все ноды кластера ssh ключи и меняем hostname:

```bash
nano machines.txt
192.168.100.10 control-plane.kubernetes.local control-plane
192.168.100.11 node-0.kubernetes.local node-0 10.200.0.0/24
192.168.100.12 node-1.kubernetes.local node-1 10.200.1.0/24


while read IP FQDN HOST SUBNET; do
	ssh -n k8s@${IP} hostname
done < machines.txt

while read IP FQDN HOST SUBNET; do
	CMD="sed -i 's/^127.0.1.1.*/127.0.1.1 ${FQDN} ${HOST}/' /etc/hosts"
	ssh -n k8s@${IP} "sudo $CMD"
	ssh -n k8s@${IP} "sudo hostnamectl set-hostname ${HOST}"
	ssh -n k8s@${IP} "sudo systemctl restart systemd-hostnamed"
done < machines.txt

while read IP FQDN HOST SUBNET; do
	ENTRY="${IP} ${FQDN} ${HOST}"
	echo "${ENTRY}" >> /etc/hosts
done < machines.txt
```


**Сертификаты**

Далее идет самое интересное, так как все компоненты кластера друг другу не доверяют, это доверие нужно построить. [[kubernetes-mtls|Создание для каждого компонента mTLS сертификата]]. После того, как мы сгенерировали сертификаты, нужно раскидать их по виртуалкам.

**На control-plane (`/var/lib/kubernetes/`):**

1) `.../ca.pem` - корневой публичный сертификат.
2) `.../ca-key.pem` - секретный ключ корневого центра.
3) `.../kubernetes.pem` - публичный серт api-сервера.
4) `.../kubernetes-key.pem` - приватный ключ api-сервера.
5) `.../service-account.pem` - публичный ключ для верификации токенов подов.
6) `.../service-account-key.pem` - приватный ключ service-account.

```bash
scp ca.pem ca-key.pem \
kubernetes.pem kubernetes-key.pem \
service-account.pem service-account-key.pem \
k8s@control-plane:/tmp/
ssh k8s@control-plane sudo mkdir -p /var/lib/kubernetes/pki/
ssh k8s@control-plane sudo mv /tmp/*.pem /var/lib/kubernetes/pki/
```

**На worker-nodes (`/var/lib/kubernetes/`)**:

1) `.../ca.pem` - корневой публичный сертификат.
2) `.../node-0.pem` - персональный паспорт первой ноды (`kubelet`).
3) `.../node-0-key.pem` - секретный приватный ключ ноды.

```bash
for host in node-0 node-1; do 
	ssh k8s@${host} sudo mkdir -p /var/lib/kubernetes/pki/; 
	scp ca.pem k8s@${host}:/tmp/; 
	scp ${host}.pem k8s@${host}:/tmp/; 
	scp ${host}-key.pem k8s@${host}:/tmp; 
	ssh k8s@${host} sudo mv /tmp/*.pem /var/lib/kubernetes/pki/; 
done
```


**Теперь можно приступать к созданию `kubeconfig`'ов:**

Каждый файл `kubeconfig` собирается последовательным выполнением **трех команд** `kubectl config`:

1. **`set-cluster`** — прописывает в конфиг адрес удаленной мастер-ноды (`192.168.100.10:6443`) и внедряет корневой сертификат `ca.pem`, чтобы компонент доверял серверу.
2. **`set-credentials`** — берет личные `.pem`-ключи конкретного компонента, кодирует их в текст (Base64) и зашивает в конфиг, чтобы сервер доверял компоненту.
3. **`set-context` + `use-context`** — склеивает адрес сервера и ключи пользователя в единый рабочий профиль (контекст).

Параметр **`--embed-certs=true`** критически важен: он заставляет `kubectl` скопировать содержимое тяжелых `.pem`-файлов **прямо внутрь создаваемого YAML-файла**. Благодаря этому конфиг становится монолитным — его можно переносить на любую машину как один файл.

>Создание конфигов для кублетов worker-нод:

```bash
for host in node-0 node-1; do
  kubectl config set-cluster kubernetes-the-hard-way \
	--certificate-authority=ca.pem \
	--embed-certs=true \
	--server=https://control-plane.kubernetes.local:6443 \
	--kubeconfig=${host}.kubeconfig

  kubectl config set-credentials system:node:${host} \
	--client-certificate=${host}.pem \
	--client-key=${host}-key.pem \
	--embed-certs=true \
	--kubeconfig=${host}.kubeconfig
	
  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${host} \
    --kubeconfig=${host}.kubeconfig
    
  kubectl config use-context default \
    --kubeconfig=${host}.kubeconfig
done
```

Конфиги создаются на админской тачке, где мы и создавали серты, `kubectl` вшивает в конфиги созданные сертификаты компонентов. Kubeconfig использует НЕ ТОЛЬКО `kubectl`, а все клиенты API. Кубернетес не знает про бинари, он знает только про пользователей, и identity (`system:node:${host}`). В kubeconfig кублета, имя пользователя в сертификате должно совпадать с названием ноды.

- **Кубконфиги** — это «сетевые паспорта» для **клиентских** программ (`scheduler`, `proxy`, `kubectl`), чтобы они могли достучаться до API-сервера по сети.
- **Файлы на диске (`.pem`)** — это «серверные лицензии» для самих **серверов** (`kube-apiserver` на мастере и `kubelet` на воркерах), чтобы они могли поднимать зашифрованные HTTPS-порты (`6443` и `10250`), а также секретные ключи для подписи токенов подов

Создание конфига для `kube-proxy`:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://control-plane.kubernetes.local:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

Создание конфига для `kube-controller-manager`:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://control-plane.kubernetes.local:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

Создание конфига для `kube-scheduler`:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://control-plane.kubernetes.local:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

Создание конфига для админа (`kubectl`):

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://control-plane.kubernetes.local:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

Отправляем конфиги на нужные машины, worker nodes:

```bash
for host in node-0 node-1; do 
	ssh k8s@${host} sudo mkdir -p /var/lib/{kube-proxy,kubelet}; 
	scp kube-proxy.kubeconfig k8s@${host}:/tmp/; 
	scp ${host}.kubeconfig k8s@${host}:/tmp/; 
	ssh k8s@${host} sudo mv /tmp/kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig; 
	ssh k8s@${host} sudo mv /tmp/${host}.kubeconfig /var/lib/kubelet/kubeconfig
done
```

Master node:

```bash
scp admin.kubeconfig \
kube-controller-manager.kubeconfig \
kube-scheduler.kubeconfig \
k8s@control-plane
```

И на админской тачке нужно закинуть админский kubeconfig в `~/.kube/config`, для того, чтобы `kubectl` понимала с чем работать.

```bash
cp admin.kubeconfig ~/.kube/config

kubectl config get-contexts # Проверяем
```


**Шифрование данных**

В etcd (хранилище кластера) данные хранятся в незашифрованном, base64 закодированном виде, поэтому кто угодно сможет увидеть секреты. Для этого нужно включить шифрование данных перед записью в etcd: 

```bash
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

Команда, для автоматического подставления переменной окружения в шаблон конфигурационного файла шифрования:

```bash
envsubst < configs/encryption-config.yaml > encryption-config.yaml
```

Сам файл есть в шаблонах, и выглядит он так:

```yaml
kind: EncryptionConfiguration
apiVersion: apiserver.config.k8s.io/v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
```

Этот конфиг читает только api-server, обычные worker ноды даже не знают, что данные зашифровываются. Поэтому его нужно отправить на master ноду:

```bash
scp encryption-config.yaml k8s@control-plane:~/
```


### Master node

**Etcd**

Переносим на мастер ноду `etcd`, `etcdctl`, `etcd.service` (директория units). Закидываем бинари в `/usr/local/bin`.

Все компоненты куба - stateless (не хранят состояние). Поэтому все данные и состояния хранятся в базе данных etcd. Настройка конфигурации etcd:

```bash
# Создаем директории для конфигурации
mkdir -p /etc/etcd /var/lib/etcd
# Читать конфигурацию сможет только root
sudo chmod 700 /var/lib/etcd
# Копируем сертификаты, чтобы база работала по https
cp ca.pem kubernetes.pem kubernetes-key.pem /etc/etcd/
# Перемещаем скопированный юнит файл, чтобы systemd его увидел
mv etcd.service /etc/systemd/system
# Просим systemd перечитать файлы с диска
sudo systemctl daemon-reload
# Включаем автозапуск etcd при старте машины
sudo systemctl enable etcd
# Запускаем базу данных в памяти прямо сейчас!
sudo systemctl start etcd
# Проверяем статус службы
systemctl status etcd
# Проверка самой базы
sudo etcdctl member list --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.pem --cert=/etc/etcd/kubernetes.pem --key=/etc/etcd/kubernetes-key.pem
```


**Control plane**

После успешного запуска базы etcd, можно перекинуть оставшиеся бинари, их unit-файлы и kubeconfig на мастер.

```bash
scp downloads/controller/kube-apiserver 
downloads/controller/kube-controller-manager 
downloads/controller/kube-scheduler 
downloads/client/kubectl 
units/kube-apiserver.service 
units/kube-controller-manager.service 
units/kube-scheduler.service 
configs/kube-scheduler.yaml 
configs/kube-apiserver-to-kubelet.yaml 
k8s@control-plane:~/
```

Все ключевые файлы control-plane складываются в `/var/lib/kubernetes`, в ней лежит все чувствительное: серты, ca, ключи, конфиги. Если утерять доступ к этой директории, то кластер скорее всего не восстановить без пересоздания компонентов.

#### Конфиги в `/etc/kubernetes/config/`

`/etc/kubernetes/config/kube-scheduler.yaml`:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
```

`/etc/kubernetes/config/kube-apiserver-to-kubelet.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
```

#### Юниты в `/etc/systemd/system/`

`/etc/systemd/system/etcd.service`:

```ini
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name=controller \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls=https://127.0.0.1:2380 \
  --listen-peer-urls=https://127.0.0.1:2380 \
  --listen-client-urls=https://127.0.0.1:2379 \
  --advertise-client-urls=https://127.0.0.1:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster=controller=https://127.0.0.1:2380 \
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
- В нашем случае написано `control-plane=https://192.168.100.10:2380`. Если бы мастеров было три, строка бы выглядела так: `master1=https://...:2380,master2=https://...:2380...`.

6. `--initial-cluster-token`
- **За что отвечает:** Секретное кодовое слово (у нас `etcd-k8s-token`). Защищает от случайного зацикливания сетей. Если в вашей домашней Wi-Fi сети кто-то рядом тоже запустит свой `etcd`, их базы не попытаются автоматически объединиться в один кластер, так как токены (пароли кластеров) не совпадут.

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
  --etcd-servers=https://127.0.0.1:2379 \
  --event-ttl=1h \
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \
  --runtime-config='api/all=true' \
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \
  --service-account-issuer=https://control-plane.kubernetes.local:6443 \
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
- **За что отвечает:** Приказывает ядру открыть порт **6443** (главный порт Kubernetes) абсолютно на всех доступных сетевых интерфейсах мастера (и на `127.0.0.1`, и на Wi-Fi адресе `192.168.100.10`). Благодаря этому вы можете управлять кластером со своего ноутбука по воздуху.

3. `--enable-admission-plugins=...,NodeRestriction,...` (Плагины допуска)
- **За что отвечает:**Admission-плагины — это судейская коллегия ядра Кубера. Когда вы просите создать под, запрос сначала проходит проверку RBAC (можно ли вам вообще создавать поды), а затем падает наAdmission-плагины. Они проверяют бизнес-логику: например, `LimitRanger` проверяет, не запросил ли под больше памяти, чем разрешено в этом namespace, а `NodeRestriction` жестко ограничивает права воркеров.

4. `--service-cluster-ip-range=10.32.0.0/24`
- **За что отвечает:** Это виртуальная подсеть, из которой Kubernetes будет выделять IP-адреса для внутренних абстракций — **Сервисов (Services)**. Эти адреса физически не существуют на сетевых картах, ядро Linux будет перехватывать их через `iptables` и перенаправлять на реальные поды. Первым адресом в этой сети автоматически станет `10.32.0.1` — внутренний IP самого API-сервера, который мы зашивали в его сертификат.

5. `--service-account-signing-key-file` и `--service-account-issuer`
- **За что отвечает:** Включают механизм генерации токенов безопасности для контейнеров (подов). API-сервер использует приватный ключ `service-account-key.pem` для криптографической подписи JWT-токенов, которые поды используют для подтверждения своей личности

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

`/etc/systemd/system/kube-scheduler.service`

```ini
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --config=/etc/kubernetes/config/kube-scheduler.yaml \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```


После раскладывания бинарников и конфигов по нужным директориям, `kube-apiserver` должен заработать, проверить можно с помощью:

```bash
sudo etcdctl member list --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.pem --cert=/etc/etcd/kubernetes.pem --key=/etc/etcd/kubernetes-key.pem
#91d7626d11b5d211, started, controller, https://127.0.0.1:2380, https://127.0.0.1:2379, false

kubectl cluster-info
#Kubernetes control plane is running at https://control-plane.kubernetes.local:6443

k8s@control-plane:~$ kubectl get componentstatuses
#NAME                 STATUS    MESSAGE   ERROR
#scheduler            Healthy   ok
#controller-manager   Healthy   ok
#etcd-0               Healthy   ok

curl --cacert ca.pem https://control-plane.kubernetes.local:6443
#{
#  "kind": "Status",
#  "apiVersion": "v1",
#  "metadata": {},
#  "status": "Failure",
#  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
#  "reason": "Forbidden",
#  "details": {},
#  "code": 403
#}
```

Если везде получили такой вывод, control-plane запустился успешно!

Если api-server не запускается, нужно глянуть логи:

```bash
sudo journalctl -u kube-apiserver.service -n 50 
```

Проблемы, которые были у меня:
1) Так как изначально unit-файлы использовал шаблонные, в них отличались пути к файлам конфигураций и сертификатов, поэтому юниты всегда нужно внимательно проверять и подгонять под себя.
2) Отличалось доменное имя кластера, а сертификат для kube-apiserver был сгенерен для других доменов. При генерации сертификата, указываются все имена, по которым будут обращаться к серверу, так как он у меня называется control-plane, нужно было добавить несколько доменов (control-plane.kubernetes.local) в команду и перегенерировать сертификат и потом раскидать его по нужным директориям.
3) При переводе базы etcd с простого http на https (mtls) необходимо добавить соответствующие флаги-указания расположения сертификатов в юнит kube-apiserver.
4) После запуска etcd, нужно проверить, оба ли порта запустились на https, потому что она может промолчать об этом, и при ошибке во флагах, запустить эндпоинт без шифрования. И важно помнить, что etcd это база данных, она сохраняет нерабочие конфиги в себе, поэтому нужно их очищать  `sudo rm -rf /var/lib/etcd` либо `etcdctl member update`!

После успешной установки control-plane, нужно заапплаить конфиг по пути `etc/kubernetes/config/kube-apiserver-to-kubelet.yaml` для правил RBAC:

```bash
kubectl apply -f kube-apiserver-to-kubelet.yaml
```


### Worker nodes

![[Pasted image 20260722235120.png]]

**Конфиг файлы**:

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
maxPods: 16
memorySwap:
  swapBehavior: NoSwap
port: 10250
resolvConf: "/etc/resolv.conf"
registerNode: true
runtimeRequestTimeout: "15m"
clusterDNS:
  - "10.32.0.10"
clusterDomain: "cluster.local"
podInfraContainerImage: "registry.k8s.io/pause:3.9"
tlsCertFile: "/var/lib/kubelet/node-0.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/node-0-key.pem"
```

`/var/lib/kube-proxy/kube-proxy-config.yaml`:

```yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
```

`/etc/cni/net.d/10-bridge.conf`:

```
{
  "cniVersion": "1.0.0",
  "name": "bridge",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [{"subnet": "10.200.0.0/24"}]
    ],
    "routes": [{"dst": "0.0.0.0/0"}]
  }
}
```

`/etc/cni/net.d/99-loopback.conf`:

```
{
  "cniVersion": "1.1.0",
  "name": "lo",
  "type": "loopback"
}
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


**Unit файлы (`/etc/systemd/system/`):**

`containerd.service`:

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

`kubelet.service`:

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

`kube-proxy.service`:

```
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Создаем конфиги и сразу переносим на ноды:

```bash
for HOST in node-0 node-1; do 
	SUBNET=$(grep ${HOST} machines.txt | cut -d " " -f 4); 
	sed "s|SUBNET|$SUBNET|g" configs/10-bridge.conf > 10-bridge.conf; 
	sed "s|SUBNET|$SUBNET|g" configs/kubelet-config.yaml > kubelet-config.yaml; 
	scp 10-bridge.conf kubelet-config.yaml k8s@${HOST}:~/; 
done
```

Копируем все бинари, нужные конфиги и юнит файлы:

```bash
for HOST in node-0 node-1; do
	scp downloads/worker/* downloads/client/kubectl \
	configs/99-loopback.conf configs/containerd-config.toml \
	configs/kube-proxy-config.yaml units/containerd.service \
	units/kubelet.service units/kube-proxy.service k8s@${HOST}:~/
done
```

Копируем все cni-плагины:

```bash
for HOST in node-0 node-1; do
	scp downloads/cni-plugins/* k8s@${HOST}:~/cni-plugins/
done
```

Устанавливаем нужные утилиты для настройки сети:

```bash
for HOST in node-0 node-1; do
	ssh k8s@${HOST} sudo apt update && sudo apt install -y socat conntrack ipset kmod
done
```

Нужно выключить раздел подкачки swap на worker nodes, это нужно для корректной работы куба:

```bash
sudo swapoff -a
sudo nano /etc/fstab # закомментировать строку со swap
```

Создаем рабочие директории, где kubelet и kubeproxy хранят свое состояние, раскидываем по ним файлы и настраиваем сеть подов:

```bash
for HOST in node-0 node-1; do
	ssh k8s@${HOST} sudo mkdir -p /etc/cni/net.d /opt/cni/bin /var/lib/kubelet /var/lib/kube-proxy /var/lib/kubernetes/ /var/run/kubernetes
	ssh k8s@${HOST} sudo mv crictl kube-proxy kubelet runc /usr/local/bin/
	ssh k8s@${HOST} sudo mv containerd containerd-shim-runc-v2 containerd-stress /usr/local/bin/
	ssh k8s@${HOST} sudo mv cni-plugins/* /opt/cni/bin/
	ssh k8s@${HOST} sudo mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/
done
```

Включаем поддержку bridge-трафика в ядре Linux:

```bash
sudo modprobe br-netfilter
echo "br-netfilter" | sudo tee -a /etc/modules-load.d/modules.conf

echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf

sudo sysctl -p /etc/sysctl.d/kubernetes.conf
```

Теперь раскидываем оставшиеся конфиги и юниты:

```bash
sudo mkdir -p /etc/containerd

sudo mv containerd-config.toml /etc/containerd/config.toml
sudo mv kubelet-config.yaml /var/lib/kubelet/
sudo mv kube-proxy-config.yaml /var/lib/kube-proxy/

sudo mv containerd.service /etc/systemd/system/
sudo mv kubelet.service /etc/systemd/system/
sudo mv kube-proxy.service /etc/systemd/system/
```

Добавляем в автозагрузку и запускаем сервисы

```bash
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

Если все сделано правильно, то ноды должны были прикрепиться к кластеру (`kubelet`'ы сами должны зарегать их в api), проверить можно так:

```bash
kubectl get nodes
#NAME     STATUS   ROLES    AGE   VERSION
#node-0   Ready    <none>   76m   v1.32.3
#node-1   Ready    <none>   75m   v1.32.3
```

#### Роуты между подами 

Так как в этом варианте CNI-плагины не используются, мы вручную настраиваем сети между подами и сервисами. В этом нам помогают конфиги сети выше (`10-bridge.conf` и `99-loopback.conf`), которые читает служба container-runtime (в нашем случае containerd), а не `kubelet`, как может показаться на первый взгляд. 

Первым делом нужно настроить kubectl (если вы этого еще не сделали) потому что кластер уже запущен, админский kubeconfig нужно положить в `~.kube/config`.

`kubelet` уже знает какой под CIDR принадлежит какой ноде, но не делает с этим ничего, для этого нужны CNI-плагины. Если под на первой worker-node попытается достучаться до пода на другой ноде, у него ничего не получится. И сделать `kubectl exec` тоже не выйдет.

Если использовать CNI плагины, вместо ручной маршрутизации подов, то необходимость конфигов (`10-bridge.conf` и `99-loopback.conf`) для containerd отпадает, но нужно будет прописывать маски сетей нод в конфиге kubelet: `podCIDR: "10.200.0.0/24"`.

В моем случае мы прямо на жесткий диск воркера в папку `/etc/cni/net.d/` вручную положили файл, где написано: `"subnet": "10.200.0.0/24"`. Когда `containerd` запускает под, он вообще не спрашивает у Kubelet диапазон сетей. Он берет этот IPAM-блок из локального файла и сам нарезает IP-адреса контейнерам. А роуты мы настроили вручную через `ip route add`. Все это делают CNI плагины под капотом.

```bash
SERVER_IP=$(grep control-plane machines.txt | cut -d " " -f 1); 
NODE_0_IP=$(grep node-0 machines.txt | cut -d " " -f 1); 
NODE_0_SUBNET=$(grep node-0 machines.txt | cut -d " " -f 4); 
NODE_1_IP=$(grep node-1 machines.txt | cut -d " " -f 1); 
NODE_1_SUBNET=$(grep node-1 machines.txt | cut -d " " -f 4);

ssh k8s@control-plane <<EOF
	sudo ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
	sudo ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF

ssh k8s@node-0 sudo ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}

ssh k8s@node-1 sudo ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
```

Ошибки с которыми столкнулся:

1) При `kubectl exec`, запрос идет сначала на мастер ноду:

```
Post "https://node-0:10250/exec/default/test-nginx-b6dfcf6bd-wfqrr/nginx?command=sh&input=1&output=1&tty=1": dial tcp: lookup node-0 on 127.0.0.53:53: server misbehaving
```

так как общего dns сервера нет, мастер не знает что за адрес у node-0 (нода, на которой размещен под и на которую идет запрос). Поэтому на мастере в `/etc/hosts` нужно объявить доменные имена нод.


**Установка coreDNS**

Чтобы поды могли между собой общаться по DNS, необходимо установить coreDNS. Для этого в конфиге `kubelet` должны быть 2 параметра `clusterDNS="10.32.0.10"` (указание адреса DNS сервера для подов) и `clusterDomain: "cluster.local"`.

Установка coreDNS производится непосредственно в сам кластер:

```bash
kubectl apply -f https://raw.githubusercontent.com/mmumshad/kubernetes-the-hard-way/master/deployments/coredns.yaml
```

Проблемы с которыми столкнулся:

1) Бесконечный цикл запросов на DNS сервер "выше". Если coreDNS не знает домен, он отправляет запрос другому серверу. В ubuntu в файле `/etc/resolv.conf` заглушка с адресом 127.0.0.53 (адрес самого coreDNS). Нужно в конфиге `kubelet` заменить `resolvConf` на `resolvConf: "/run/systemd/resolve/resolv.conf"`.

После этого поды смогут общаться между собой по доменным именам. 

Проверки что все работает и кластер полностью готов и установлен:

```bash
kubectl create deploy test-nginx --image=nginx --replicas=2

kubectl port-forward --address 0.0.0.0 test-nginx-b6dfcf6bd-wfqrr 8080:80

kubectl expose deployment test-nginx --port=80 --type=NodePort

kubectl exec -it <po_name> -- bash

curl test-nginx # Должно отработать 
```