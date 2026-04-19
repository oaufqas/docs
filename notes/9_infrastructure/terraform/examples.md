#### Развертывание [[architecture|Kubernetes cluster]] с IngressController nginx и cert-manager в [[managed-k8s|Yandex Cloud]] с помощью Terraform

```main.tf
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
    helm = {
      source = "hashicorp/helm"
    }
  }
}


provider "yandex" {
  token     = var.token
  cloud_id  = var.cloud_id
  folder_id = var.folder_id
  zone      = var.zone
}

provider "helm" {
  kubernetes = {
    host                   = yandex_kubernetes_cluster.k8s-cluster.master[0].external_v4_address
    cluster_ca_certificate = yandex_kubernetes_cluster.k8s-cluster.master[0].cluster_ca_certificate

    token = var.token
  }
}

resource "yandex_vpc_network" "k8s-network" {
  name = "k8s-network"
}

resource "yandex_vpc_address" "ingress-static-ip" {
  name = "ingress-ip"
  external_ipv4_address {
    zone_id = var.zone
  }
}

resource "yandex_vpc_subnet" "k8s-subnet" {
  name           = "k8s-subnet"
  network_id     = yandex_vpc_network.k8s-network.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

resource "yandex_iam_service_account" "k8s-sa" {
  name        = "k8s-account"
  description = "Service account for Kubernetes cluster"
}

resource "yandex_resourcemanager_folder_iam_member" "k8s-roles" {
  for_each  = toset(["editor", "container-registry.images.puller"])
  folder_id = var.folder_id
  role      = each.key
  member    = "serviceAccount:${yandex_iam_service_account.k8s-sa.id}"
}

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
}

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
  repository       = "https://charts.jetstack.io"
  chart            = "cert-manager"
  namespace        = "cert-manager"
  create_namespace = true
  version          = "v1.14.0"
  
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

output "ingress_external_ip" {
  value = yandex_vpc_address.ingress-static-ip.external_ipv4_address[0].address
}

```

```variables.tf
variable "token" {}
variable "cloud_id" {}
variable "folder_id" {}
variable "zone" {}
```

### Создание виртуальной машины в yandex cloud

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