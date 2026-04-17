### Providers (Провайдеры)

Провайдеры — это «переводчики, обстракция» между кодом Terraform и API конкретного сервиса. Сам Terraform ничего не знает о том, как создать сервер в Yandex Cloud или базу данных в AWS.

- **Как это работает:** Когда вы пишете код, провайдер скачивается как отдельный бинарный файл (`terraform init`). Он принимает команды от Terraform и отправляет соответствующие запросы в облако.
- **Типы:** Есть официальные (от HashiCorp), партнерские (от самих облаков) и комьюнити-провайдеры (для экзотических сервисов).
- **Зачем:** Позволяют управлять всем из одного места — от «железа» в облаке до пользователей в GitHub или Cloudflare.

Документация на популярных провайдеров:

- **[Yandex cloud](https://yandex.cloud/ru/docs/terraform/resources/kubernetes_cluster)**
- **[AWS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)** (vpn)
- **[Other](https://registry.terraform.io/browse/providers)** (vpn)