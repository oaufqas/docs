#### Cli yandex (yc) - собственная командная оболочка яндекса, для управления облаком

Установка на Linux

```bash
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash

yc init --username=<email_addr>
```

Подключение уже созданного кластера с помощью yandex cli

```bash
yc managed-kubernetes cluster get-credentials --id cat5541680n5sih1vggs --external
```

Удаление кластера

```bash
yc managed-kubernetes cluster delete --name my-k8s-cluster
```