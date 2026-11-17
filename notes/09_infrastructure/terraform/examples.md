### Yc-provider & helm

Настройка провайдеров.

```hcl
terraform {
  required_providers { # Объявляем провайдеры, которые будем использовать
    yandex = {
      source = "yandex-cloud/yandex" 
    }
    helm = {
      source = "hashicorp/helm"
    }
  }
}

provider "yandex" { # Все данные берутся с веб-панели yandex-cloud
  token     = var.token
  cloud_id  = var.cloud_id
  folder_id = var.folder_id
  zone      = var.zone
}

provider "helm" {
  kubernetes = {
    host                   = yandex_kubernetes_cluster.k8s-cluster.master[0].external_v4_address # Указываем адрес кластера
    
    cluster_ca_certificate = yandex_kubernetes_cluster.k8s-cluster.master[0].cluster_ca_certificate # Указываем сертификат кластера

    token = var.token
  }
}
```


Развертывание [[architecture|Kubernetes cluster]] в [[managed-k8s|Yandex Cloud]] с помощью Terraform.


```hcl
resource "yandex_vpc_network" "k8s-network" { # Создаем сеть в яндексе
  name = "k8s-network"
}

resource "yandex_vpc_address" "ingress-static-ip" { # Получаем белый ip для ingress
  name = "ingress-ip"
  external_ipv4_address {
    zone_id = var.zone # Зоны могут быть A,B,C,D, уточняется в веб-панели
  }
}


# Создаем подсеть ссылаясь на созданную сеть k8s-network
resource "yandex_vpc_subnet" "k8s-subnet" { 
  name           = "k8s-subnet"
  network_id     = yandex_vpc_network.k8s-network.id
  v4_cidr_blocks = ["192.168.10.0/24"]
  zone           = var.zone
}


# Создаем сервисный аккаунт в яндексе, который будет устанавливать кластер
resource "yandex_iam_service_account" "k8s-sa" { 
  name        = "k8s-account"
  description = "Service account for Kubernetes cluster"
}


# Выдаем циклом роли для сервисного аккаунта чтобы были права на создание
resource "yandex_resourcemanager_folder_iam_member" "k8s-roles" {
  for_each  = toset(["editor", "container-registry.images.puller"])
  folder_id = var.folder_id
  role      = each.key
  member    = "serviceAccount:${yandex_iam_service_account.k8s-sa.id}"
}


# Создаем кластер
resource "yandex_kubernetes_cluster" "k8s-cluster" {
  name       = "k8s-cluster"
  network_id = yandex_vpc_network.k8s-network.id

  master {
    zonal {
      zone      = yandex_vpc_subnet.k8s-subnet.zone
      subnet_id = yandex_vpc_subnet.k8s-subnet.id
    }
    public_ip = true
  }

  service_account_id      = yandex_iam_service_account.k8s-sa.id
  node_service_account_id = yandex_iam_service_account.k8s-sa.id

  depends_on = [
    yandex_resourcemanager_folder_iam_member.k8s-roles
  ]
}

resource "yandex_kubernetes_node_group" "nodes" {
  cluster_id = yandex_kubernetes_cluster.k8s-cluster.id
  name       = "nodes"

  instance_template {
    platform_id = "standard-v3"

    resources {
      memory = 4
      cores  = 2
    }

    boot_disk {
      type = "network-hdd"
      size = 32
    }

    scheduling_policy {
      preemptible = true
    }

    network_interface {
      nat        = true
      subnet_ids = [yandex_vpc_subnet.k8s-subnet.id]
    }
  }

  scale_policy {
    fixed_scale {
      size = 2
    }
  }

  allocation_policy {
    location {
      zone = var.zone
    }
  }
}
```


Установка IngressController nginx и cert-manager в готовый кластер с помощью `helm_release`


```hcl
resource "helm_release" "ingress-nginx" {
  name             = "ingress-nginx"
  repository       = "https://kubernetes.github.io/ingress-nginx"
  chart            = "ingress-nginx"
  namespace        = "ingress-nginx"
  create_namespace = true

  set = [
    {
      name  = "controller.service.loadBalancerIP"
      value = yandex_vpc_address.ingress-static-ip.external_ipv4_address[0].address
    }
  ]
  depends_on = [
    helm_release.cert_manager
  ]
}

resource "helm_release" "cert_manager" {
  name             = "cert-manager"
  repository       = "oci://quay.io/jetstack/charts"
  chart            = "cert-manager"
  namespace        = "cert-manager"
  create_namespace = true
  version          = "v1.16.1"

  set = [
    {
      name  = "installCRDs"
      value = "true"
    }
  ]
  depends_on = [
    yandex_kubernetes_cluster.k8s-cluster
  ]
}

resource "helm_release" "nfs_server" {
  name             = "nfs-server-provisioner"
  repository       = "https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner"
  chart            = "nfs-server-provisioner"
  namespace        = "nfs-system"
  create_namespace = true

  set = [{
    name  = "persistence.enabled"
    value = "true"
    },
    {
      name  = "persistence.size"
      value = "4Gi"
    },
    {
      name  = "persistence.storageClass"
      value = "yc-network-hdd"
    },
    {
      name  = "storageClass.name"
      value = "nfs"
  }]
}

output "ingress_external_ip" {
  value = yandex_vpc_address.ingress-static-ip.external_ipv4_address[0].address
}
```


Создание виртуальной машины в yandex cloud


```hcl
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}


provider "yandex" {
  token     = var.token
  cloud_id  = var.cloud_id
  folder_id = var.folder_id
  zone      = var.zone
}


# 1. Находим образ ОС (Динамически узнаем актуальный ID)
data "yandex_compute_image" "ubuntu" {
  family = "ubuntu-2204-lts"
}

resource "yandex_compute_instance" "vm-1" {
  name        = "linux-vm"
  platform_id = "standard-v1"
  zone        = var.zone

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = data.yandex_compute_image.ubuntu.id # Ссылка на Data Source
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet_1.id
    nat       = true # ОБЯЗАТЕЛЬНО: дает публичный IP для входа по SSH
  }

  metadata = {
    # 'ubuntu' - это стандартный логин в этом образе. 
    # file() читает ваш локальный публичный ключ.
    ssh-keys = "ubuntu:${file("~/.ssh/id_ed25519.pub")}"
  }
}

resource "yandex_vpc_network" "network_1" {
  name = "network-1"
}

resource "yandex_vpc_subnet" "subnet_1" {
  name           = "subnet-1"
  zone           = var.zone
  network_id     = yandex_vpc_network.network_1.id
  v4_cidr_blocks = ["10.5.0.0/24"]
}

# Чтобы узнать IP машины, не заходя в консоль Яндекса
output "vm_public_ip" {
  value = yandex_compute_instance.vm-1.network_interface.0.nat_ip_address
}

```



---

#### Proxmox provider & local generating file outputs.

Создание тачек для кластера из списка.

```hcl
locals { # Объявление локальных переменных, карта будущей инфраструктуры
    k8s_nodes = {
        "control-plane" = { id = 110, ip = "192.168.100.10/24" }
        "node-0" = { id = 111, ip = "192.168.100.11/24" }
        "node-1" = { id = 112, ip = "192.168.100.12/24" }
        "node-2" = { id = 113, ip = "192.168.100.13/24" }
    }
}  
  

resource "proxmox_virtual_environment_vm" "k8s_cluster" {
    for_each = local.k8s_nodes # Запускает цикл. Ресурс повторится столько раз, сколько записей в словаре
    
    name = each.key # Имя виртуалки в Proxmox станет равна ключу словаря
    node_name = "pve" # Имя сервера proxmox
    vm_id = each.value.id # Имя виртуалки

    clone { # Откуда копировать виртуалки, в нашем случае template с id 100
        vm_id = 100
    }  

    cpu { # Количество ядер для вируалки
        cores = 1
        type = "host"
    }  

    memory { # Объем памяти для виртуалки
        dedicated = 2048
    }  

    network_device { # Выбор сетевой карты 
        bridge = "vmbr0"
    }  

    initialization { # Cloud-init параметры
        ip_config {
            ipv4 {
                address = each.value.ip # Берем ipv4 из словаря locals
                gateway = var.gateway # Шлюз берем из переменных
            }
        }  

        dns { # DNS сервера виртуалки берем из переменных (список)
            servers = var.dns_servers
        }  

        user_account { # Создание пользователя, пароль, ssh в authorized_keys, можно несколько ключей добавить
            username = "k8s"
            password = "k8spasswd"
            keys = [
                "${var.ssh_key}"
            ]
        }
    }
}
```

Скачивание ISO-образов из интернета напрямую

```hcl
resource "proxmox_virtual_environment_file" "ubuntu_iso" {
  content_type = "iso"
  datastore_id = "local" # Образы ISO всегда хранятся в local
  node_name    = "pve"

  source_file {
    url = "https://ubuntu.com" # Главный аргумент, pve скачает образ используя wget
  }
}

# proxmox_virtual_environment_file под капотом работает не по API токену подключения (8006 port), а по ssh (22 port), поэтому для работы этого ресурса нужно объявить доп параметры в настройке провайдера:
ssh { 
	agent = true 
	username = "root" 
	password = psswd # или
	private_key = file("~/.ssh/id_rsa")
  }
```

Динамическое создание дисков в LVM-Thin

```hcl
resource "proxmox_virtual_environment_disk" "data_disk" {
  datastore_id = "local-lvm" # Наш пул LVM-Thin
  node_name    = "pve"
  size         = 50          # Размер в Гигабайтах
  interface    = "scsi1"     # Подключаем как быстрый SCSI VirtIO диск
  vm_id        = 101
  
  file_format  = "raw"       # Для LVM-Thin стандарт строго RAW
}
```

Создание LXC-контейнеров (`proxmox_virtual_environment_container`)

```hcl
data "proxmox_file" "ubuntu_container_template" {
    node_name    = "pve"
    datastore_id = "local"
    content_type = "vztmpl"
    file_name    = "ubuntu-24.04-standard_24.04-2_amd64.tar.zst"

}
  

resource "proxmox_virtual_environment_container" "dns-resolver" {
    node_name = "pve"
    vm_id = 120
    unprivileged = true

    features {
        nesting = true
    }

    initialization {
        hostname = "cluster-dns-resolver"

        dns {
            servers = [ "1.1.1.1", "8.8.8.8" ]
        }

        ip_config {
            ipv4 {
                address = "${var.dns_servers[0]}/24"
                gateway = var.gateway
            }
        }

        user_account {
            keys = ["${var.ssh_key}"]
            password = var.password
        }
    }

    network_interface {
        name = "eth0"
        bridge = "vmbr0"
    }

    disk {
        datastore_id = "local-lvm"
        size         = 6
    }

    cpu {
        cores = 1
    }

    memory {
        dedicated = 512
    }

    operating_system {
        template_file_id = data.proxmox_file.ubuntu_container_template.id
        type = "ubuntu"
    }
}
```


Провайдер `local`, ресурс `local_file` со сложной интерполяцией кода для inventory файла (outputs)


```hcl
resource "local_file" "ansible_inventory" {
  filename = "${path.module}/../ansible-k8s/inventory/machines.ini"
  
  content  = <<EOF
[masters]
%{ for name, node in local.k8s_nodes ~}
%{ if name == "control-plane" ~}
${name} ansible_host=${split("/", node.ip)[0]} ansible_user=${var.user} ansible_sudo_pass=${var.password}
%{ endif ~}
%{ endfor ~}

[workers]
%{ for name, node in local.k8s_nodes ~}
%{ if name != "control-plane" ~}
${name} ansible_host=${split("/", node.ip)[0]} ansible_user=${var.user} pod_cidr=10.200.${tonumber(split("-", name)[1])}.0/24 ansible_sudo_pass=${var.password}
%{ endif ~}
%{ endfor ~}

[k8s:children]
masters
workers
EOF

  depends_on = [proxmox_virtual_environment_vm.k8s_cluster]
}
```

1. **Конструкция `%{ for name, node in local.k8s_nodes ~}` ... `%{ endfor ~}`**  
    Это встроенный цикл Terraform для генерации строк. Знак тильды `~` на конце тегов критически важен — он приказывает парсеру Terraform съедать (удалять) лишние переносы строк, чтобы ваш `machines.ini` не содержал пустых пробелов и выглядел спартански аккуратно.
2. **Фильтр условий `%{ if name != "control-plane" ~}`**  
    Мы в один проход проверяем имя ключа. Если имя не равно мастеру, мы отправляем эту ноду в секцию `[workers]`. Если вам завтра потребуется добавить `node-2`, `node-3` в `locals`, цикл сам подхватит их, и они автоматически допишутся в секцию воркеров.
3. **Динамический расчет `pod_cidr` на основе имени:**  
    `10.200.${tonumber(split("-", name)[1])}.0/24`
    - Функция `split("-", name)` берет имя `node-0` и режет его по дефису, превращая в список `["node", "0"]`.
    - Конструкция `[1]` забирает второй элемент — индекс `"0"`.
    - Функция `tonumber(...)` превращает текст `"0"` в честную цифру `0` для работы математики подсетей Кубера.  
        _В итоге для ноды `node-5` подсеть подов на лету рассчитается как `10.200.5.0/24` полностью автоматически._
4. **Сжатие IP через `split("/", node.ip)[0]`**  
    Так как в `locals` мы храним IP с маской подсети для Cloud-Init (`192.168.100.11/24`), для Ansible маска `/24` сломает SSH-подключение. Эта функция режет строку по слэшу и забирает чистый IP-адрес.