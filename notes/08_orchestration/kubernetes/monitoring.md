## Что вообще можно мониторить в кластере?

В первую очередь это control-plane, api server и подключаемые к нему компоненты: etcd, controller manager, scheduler, kubelet.

Для мониторинга рабочих узлов, универсального решения нет, самым популярным решением является запуск [[exporters|node-exporter]] на каждой ноде.

Информацию о состоянии запущенных pods предоставляет системный комонент - kubelet, запускаемый на каждой worker node kubernetes cluster.

![[Pasted image 20260525201105.png]]

Также можно и нужно мониторить сами пользовательские приложения. Это можно делать путем модификации приложения на самостоятельное экспортирование метрик в promql формате по pull-модели, приложение запускает web-server, один из эндпоинтов (например /metrics) отдает метрики коллектору. Либо написать exporter, который будет взаимодействовать с приложением и коллектором.

В control-plane kubernetes все компоненты отдают метрики в prometheus text формате.

Пример конфигурационного файла:

```yaml
scrape_configs:
- job_name: vmdb
  static_configs:
  - targets: ['127.0.0.1:8428']
- job_name: k8s-endpoints
  kubernetes_sd_configs:
  - role: endpoints
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
```

Самый простой способ сбора метрик - перечислить все нужные targets для сбора в конфигурационном файле. Но для пользовательских приложений это будет неудобно, а то и невозможно из-за постоянного скалирования подов. Для этого нужен **Service discovery**, он решает задачу автоматического отслеживания изменений в k8s. Он запускает сбор метрик с добавленных и останавливает для удаленных в targets.


Relabeling configs - это набор правил, позволяющий выполнять различные операции над labels. Каждое из правил выполняется последовательно и зависит от результатов предыдущего.

```yaml
ralabel_configs:
- source_labels: [__meta_kubernetes_node_name]
  action: replace
  modulus: 123
  regex: (.+).mydomain
  replacement: $1
  separator: ;
  terget_label: host
```

![[Pasted image 20260525210832.png]]