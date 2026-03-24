
Как работает `kubectl` (Теория)

**kubectl** — это не «пульт управления» серверами напрямую. Это **HTTP-клиент**, который отправляет команды в **kube-apiserver**.

1. **Запрос:** Ты пишешь команду (например, `get pods`).
2. **Аутентификация:** `kubectl` берет твой сертификат из `~/.kube/config`, чтобы доказать API-серверу, что это ты.
3. **Декларативность:** В Kubernetes ты не говоришь «запусти процесс», ты говоришь «я хочу, чтобы состояние кластера соответствовало этому описанию». API-сервер записывает твое желание в базу **etcd**, а контроллеры начинают подгонять реальность под это описание.

### Основные команды kubectl

##### Базовые команды

| Команда                         | Описание                                                   |
| ------------------------------- | ---------------------------------------------------------- |
| `kubectl version`               | Версия клиента и сервера                                   |
| `kubectl cluster-info`          | Информация о кластере (где api-server)                     |
| `kubectl get nodes`             | Список всех узлов (нод) кластера                           |
| `kubectl get all`               | Получить все объекты в текущем namespace                   |
| `kubectl explain pod`           | Документация по объекту (полезно для написания манифестов) |
| `kubectl get componentstatuses` | Получить статусы компонентов кластера                      |

##### Работа с ресурсами (get)

| Команда                         | Описание                                |
| ------------------------------- | --------------------------------------- |
| `kubectl get pods`              | Список подов                            |
| `kubectl get pods -o wide`      | Поды с IP и нодами                      |
| `kubectl get pods -w`           | Следить за изменениями (watch)          |
| `kubectl get deploy`            | Список деплойментов                     |
| `kubectl get svc`               | Список сервисов                         |
| `kubectl get ing`               | Список ingress                          |
| `kubectl get cm`                | ConfigMap                               |
| `kubectl get secrets`           | Секреты                                 |
| `kubectl get pv`                | PersistentVolume                        |
| `kubectl get pvc`               | PersistentVolumeClaim                   |
| `kubectl get ns`                | Namespace                               |
| `kubectl get all -A`            | Все объекты во всех namespace (-A)      |
| `kubectl get pods -l app=nginx` | По лейблам (label selector)             |
| `kubectl get cluster-info`      | Получить информацию где запущен кластер |

##### Создание и применение (apply/create)

|Команда|Описание|
|---|---|
|`kubectl apply -f file.yaml`|Применить манифест (создать или обновить)|
|`kubectl apply -f ./dir/`|Применить все манифесты в папке|
|`kubectl create -f file.yaml`|Создать объект (устаревший вариант)|
|`kubectl run nginx --image=nginx`|Создать под (только для тестов)|
|`kubectl create deploy nginx --image=nginx`|Создать deployment (только для тестов)|
|`kubectl create ns my-namespace`|Создать namespace|

##### Удаление (delete)

|Команда|Описание|
|---|---|
|`kubectl delete pod nginx`|Удалить под|
|`kubectl delete -f file.yaml`|Удалить объекты из манифеста|
|`kubectl delete deploy nginx`|Удалить deployment|
|`kubectl delete all --all`|Удалить ВСЕ объекты в namespace (осторожно!)|
|`kubectl delete ns my-namespace`|Удалить namespace (удалит всё внутри)|

##### Детальная информация (describe)

|Команда|Описание|
|---|---|
|`kubectl describe pod nginx`|Детальная информация о поде (события, статус)|
|`kubectl describe node node1`|Информация о ноде (ресурсы, поды на ней)|
|`kubectl describe deploy nginx`|Статус деплоймента (replicas, strategy)|
|`kubectl describe svc nginx`|Детали сервиса (endpoints, selector)|

##### Логи и доступ (logs/exec)

|Команда|Описание|
|---|---|
|`kubectl logs nginx`|Логи пода|
|`kubectl logs -f nginx`|Следить за логами (-f = follow)|
|`kubectl logs nginx -c container1`|Логи конкретного контейнера в поде|
|`kubectl logs --tail=100 nginx`|Последние 100 строк|
|`kubectl logs -l app=nginx`|Логи всех подов с лейблом|
|`kubectl exec -it nginx -- sh`|Зайти в контейнер (интерактивный shell)|
|`kubectl exec nginx -- ls -la`|Выполнить команду без интерактива|
|`kubectl cp /local/file.txt nginx:/tmp/`|Скопировать файл в под|

##### Управление обновлениями (rollout)

|Команда|Описание|
|---|---|
|`kubectl rollout status deploy/nginx`|Статус обновления|
|`kubectl rollout history deploy/nginx`|История обновлений|
|`kubectl rollout undo deploy/nginx`|Откат к предыдущей версии|
|`kubectl rollout restart deploy/nginx`|Перезапустить поды (без изменения образа)|
|`kubectl set image deploy/nginx nginx=nginx:1.20`|Обновить образ в deployment|

##### Масштабирование (scale)

|Команда|Описание|
|---|---|
|`kubectl scale deploy/nginx --replicas=5`|Масштабировать до 5 реплик|
|`kubectl autoscale deploy/nginx --min=2 --max=10 --cpu-percent=80`|Автомасштабирование (HPA)|

##### Проброс портов (port-forward)

|Команда|Описание|
|---|---|
|`kubectl port-forward pod/nginx 8080:80`|Пробросить локальный 8080 на порт 80 пода|
|`kubectl port-forward svc/nginx 8080:80`|Пробросить на сервис|
|`kubectl port-forward deploy/nginx 8080:80`|Пробросить на деплоймент|

##### Лейблы и аннотации

|Команда|Описание|
|---|---|
|`kubectl label pod nginx app=web`|Добавить лейбл|
|`kubectl label pod nginx app-`|Удалить лейбл (минус в конце)|
|`kubectl annotate pod nginx description="my app"`|Добавить аннотацию|

##### Namespace

|Команда|Описание|
|---|---|
|`kubectl get ns`|Список namespace|
|`kubectl config set-context --current --namespace=dev`|Переключить namespace по умолчанию|
|`kubectl get pods -n kube-system`|Поды в конкретном namespace|
|`kubectl get pods --all-namespaces`|Поды во всех namespace (коротко: `-A`)|

##### Работа с манифестами (generate/view)

|Команда|Описание|
|---|---|
|`kubectl get deploy nginx -o yaml`|Вывести манифест объекта|
|`kubectl get deploy nginx -o yaml --export`|Экспорт (без статуса)|
|`kubectl get pods -o json`|В формате JSON|
|`kubectl create deploy nginx --image=nginx --dry-run=client -o yaml`|Сгенерировать манифест без создания|


##### Ресурсы и лимиты

|Команда|Описание|
|---|---|
|`kubectl top nodes`|Использование ресурсов на нодах|
|`kubectl top pods`|Использование ресурсов подами|
|`kubectl describe quota`|Квоты namespace|

##### Команда `run`

| Команда                                                                                       | Описание                                                  |
| --------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `kubectl run my-test --image=nginx`                                                           | Простой запуск (посмотреть, работает ли образ)            |
| `kubectl run backend --image=my-app:1.0 --env="DB_URL=postgres://db:5432" --env="DEBUG=true"` | Запуск с переменными окружения                            |
| `kubectl run -it --rm debug-pod --image=busybox -- sh`                                        | Интерактивный запуск (зайти внутрь и попинговать сеть)    |
| `kubectl run my-pod --image=nginx --port=80 --dry-run=client -o yaml > pod.yaml`              | Генерация YAML (самый полезный прием для профи)           |
| `--dry-run=client`                                                                            | Не отправлять запрос в кластер, просто проверить локально |
| `-o yaml`                                                                                     | Вывести результат в формате YAML                          |

##### Очистка

|Команда|Описание|
|---|---|
|`kubectl delete pod --all`|Удалить все поды в namespace|
|`kubectl delete --all pods`|То же самое|
|`kubectl delete -f ./manifests/`|Удалить всё по папке|
|`kubectl drain node1 --ignore-daemonsets`|Вывести ноду из эксплуатации (подготовка к удалению)|
|`kubectl cordon node1`|Запретить новые поды на ноду|
|`kubectl uncordon node1`|Разрешить поды обратно|

##### Шпаргалка по часто используемым комбинациям

|Задача|Команда|
|---|---|
|Узнать, почему под не запускается|`kubectl describe pod <name>`|
|Посмотреть логи упавшего пода|`kubectl logs <name> --previous`|
|Обновить образ без манифеста|`kubectl set image deploy/nginx nginx=nginx:1.20`|
|Посмотреть, какие поды на какой ноде|`kubectl get pods -o wide`|
|Пробросить порт для отладки|`kubectl port-forward svc/my-app 8080:80`|
|Выйти из зависшего пода|`kubectl delete pod <name> --force --grace-period=0`|
|Посмотреть все события в кластере|`kubectl get events --sort-by='.lastTimestamp'`|
|Скопировать файл из пода|`kubectl cp <pod>:/path/to/file ./local-file`|