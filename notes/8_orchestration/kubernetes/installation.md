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

####  Примеры конфигурационных файлов с объяснениями [[master-nodes|есть тут]]


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

После успешной установки control-plane (можно и позже), нужно зааплаить конфиг по пути `etc/kubernetes/config/kube-apiserver-to-kubelet.yaml` для правил RBAC:

```bash
kubectl apply -f kube-apiserver-to-kubelet.yaml
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
maxPods: 32
memorySwap:
  swapBehavior: NoSwap
port: 10250
resolvConf: "/run/systemd/resolve/resolv.conf"
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
	ssh k8s@${HOST} sudo mv cni-plugins/* 2
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
2) Сервисы не резолвятся в подах:

```bash
$ nslookup test-nginx.svc.cluster.local 
Server: 10.32.0.10 
# Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local 

# nslookup: can't resolve 'test-nginx.svc.cluster.local' command terminated with exit code 1
```

Решением стало поменять домен кластера в kubelet-config. Настроить `/etc/hosts` на всех нодах и добавить домены к мастеру в сертификате, перевыпустить сертификат. Команды, которые помогают отдебажить:

```bash
kubectl edit configmap coredns -n kube-system # Проверка конфига
kubectl rollout restart deployment coredns -n kube-system # После каждого изменения рестартать
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50 # Проверка логов
kubectl get endpoints kube-dns -n kube-system
kubectl get endpoints <svc>

kubectl run busybox --image=busybox:1.28 --restart=Never -- sleep 3600 # Проверки утилитами изнутри подов
kubectl exec -ti busybox -- nslookup kube-dns.kube-system.svc.cluster.local
kubectl exec -it busybox -- wget https://10.32.0.1 --no-check-certificate
kubectl exec -it busybox -- cat /etc/resolv.conf # Проверка конфигурации coreDNS


kubectl exec -it busybox -- nslookup kubernetes.default.svc.cluster.local 10.200.1.5

# Если этот запрос СРАБОТАЛ (вернул 10.32.0.1): Значит, CoreDNS абсолютно здоров. Проблема на 100% кроется в kube-proxy, который не может перенаправить трафик с виртуального IP 10.32.0.10 на реальный под.
# Если НЕ сработал: CoreDNS полностью изолирован на уровне сети воркеров
```

В конфигурационном файле `/var/lib/kubelet/kubelet-config.yaml` в поле `clusterDomain` по мировому стандарту должно быть жестко прописано строго **`cluster.local`** (в кавычках).

```yaml
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
```

За что отвечает этот параметр:

Когда под пытается отрезолвить короткое имя (например, вы пишете `nslookup test-nginx`), Kubelet, используя значение `clusterDomain`, автоматически собирает для пода суффиксы поиска (строку `search` в файле `/etc/resolv.conf` внутри контейнера).  
Он дописывает туда: `default.svc.cluster.local`, `svc.cluster.local`, `cluster.local`.

---

После этого поды смогут общаться между собой по доменным именам. 

Проверки что все работает и кластер полностью готов и установлен:

```bash
kubectl create deploy test-nginx --image=nginx --replicas=2

kubectl port-forward --address 0.0.0.0 test-nginx-b6dfcf6bd-wfqrr 8080:80

kubectl expose deployment test-nginx --port=80 --type=NodePort

kubectl exec -it <po_name> -- bash

curl test-nginx # Должно отработать 
```


---

#### Переезд на CNI-plugin calico

Ручной вариант настройки сети хорош для понимания работы сети в kubernetes. Но он сложно масштабируем, если добавилась новая worker-node, нужно изменять настройки на всех уже имеющихся. Поэтому для начала нужно полностью очистить папку `/etc/cni/net.d/` на воркерах. CNI-plugin самостоятельно создаст в ней нужные файлы. Удалить ручные маршруты (`ip route add...`), можно просто перезагрузить сервер, данные хранятся в оперативной памяти. **Удалить виртуальный мост `cni0`:**  
Старый мост, который создавала утилита `bridge`, больше не нужен. 

```bash
sudo ip link set cni0 down
sudo ip link delete cni0
```

Настройка конфигов и становка CNI:

- **Вернуть и жестко зафиксировать `podCIDR` в `/var/lib/kubelet/kubelet-config.yaml`:**  
	Теперь этот флаг становится **обязательным**. Kubelet должен четко рапортовать Мастеру, каким диапазоном сети он владеет, чтобы CNI-плагин мог прочитать это из API-сервера и построить правильный тоннель.
	- _На Воркере 1:_ `podCIDR: "10.200.0.0/24"`
	- _На Воркере 2:_ `podCIDR: "10.200.1.0/24"`
- **Проверить пути к CNI в `containerd`:**  
	Убедитесь, что в `/etc/containerd/config.toml` рантайм по-прежнему смотрит в стандартные папки: `bin_dir = "/opt/cni/bin"` и `conf_dir = "/etc/cni/net.d"`. CNI-плагины будут подкладывать свои бинарники именно туда.

Установка манифеста:

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Изменение CIDR для подов:  

Найдите в файле `calico.yaml` переменную `CALICO_IPV4POOL_CIDR`. Она должна совпадать с CIDR, который мы использовали для сети подов при настройке кластера (это `10.200.0.0/16`). Если CIDR отличается, замените его:

```bash
sed -i 's|192.168.0.0/16|10.200.0.0/16|g' calico.yaml
```

Применияем манифест в кластер:

```bash
kubectl apply -f calico.yaml
```

Поды Calico должны запуститься в пространстве имен `kube-system`. Для проверки статуса используйте команду:

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
```

Все поды `calico-node` должны быть в состоянии `Running`.