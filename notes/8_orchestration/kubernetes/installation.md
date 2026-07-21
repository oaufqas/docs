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

Для ручной установки изначально необходимо настроить минимум 3 сервера (или виртуальные машины), которые будут находиться в одной локальной сети и видеть друг друга. В моем примере виртуалки накатаны в домашнем [[notes/10_virtualization/proxmox/basics|proxmox]], и есть LXC контейнер для администрирования виртуалок: 

```
LXC Container (kubectl, ansible, terraform) - 192.168.100.111/24
control-plane (etcd, apiserver, sheduler, controller-manager) - 192.168.100.10/24
node-0 (kubelet, kubeproxy, containerd) - 192.168.100.11/24
node-1 (kubelet, kubeproxy, containerd) - 192.168.100.12/24
Proxmox server - 192.168.100.1 (default gateway), 192.168.0.167.
```

После создания виртуалок и настройки сети (у нод желательно должны быть статические ip и обязательно разные имена хостов), нужно выключить раздел подкачки swap, потому что kubernetes с ним не будет корректно работать:

```bash
sudo swapoff -a
sudo nano /etc/fstab # закомментировать строку со swap
```

Необходимо установить все нужные бинари. Для удобства можно скопировать [репозиторий с шаблонами файлов и бинарниками](https://github.com/kelseyhightower/kubernetes-the-hard-way). 

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

**На control-plane (`/var/lib/kubernetes/pki/`):**

1) `.../ca.pem` - корневой публичный сертификат.
2) `.../ca-key.pem` - секретный ключ корневого центра.
3) `.../kubernetes.pem` - публичный серт api-сервера.
4) `.../kubernetes-key.pem` - приватный ключ api-сервера.
5) `.../service-account.pem` - публичный ключ для верификации токенов подов.
6) `.../service-account-key.pem` - приватный ключ service-account.

```bash
scp ca.pem ca-key.pem \
kubernetes.pem kubernetes-key.pem \
service-account.pem service-account.pem \
k8s@control-plane:/tmp/
ssh k8s@control-plane sudo mkdir -p /var/lib/kubernetes/pki/
ssh k8s@control-plane sudo mv /tmp/*.pem /var/lib/kubernetes/pki/
```

**На worker-nodes (`/var/lib/kubernetes/pki/`)**:

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
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom) | base64
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


**Etcd**

Все компоненты куба - stateless (не хранят состояние). Поэтому все данные и состояния хранятся в базе данных etcd.