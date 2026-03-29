**[[notes/6_ci-cd/argo-cd/installation|ArgoCD]]** — это ==инструмент для **GitOps**==. Если Helm — это «пакетный менеджер», то ArgoCD — это «автопилот», который следит, чтобы то, что лежит в твоем Git, на 100% совпадало с тем, что запущено в Kubernetes.

В обычном подходе (CI/CD) ты пишешь скрипт, который делает `helm upgrade`. В GitOps с ArgoCD ты вообще не трогаешь консоль.

Как это работает (Принцип «Стяжки»)

1. **Git как единственный источник истины:** Все твои Helm-чарты или YAML-манифесты лежат в репозитории (например, в GitHub).
2. **ArgoCD внутри кластера:** Это контроллер, который постоянно «смотрит» в твой Git и в твой Kubernetes.
3. **Синхронизация (Sync):**
    - Если ты поправил в Git количество реплик с 2 на 5 — ArgoCD увидит разницу и **сам** применит изменения в кластере.
    - Если кто-то вручную зашел в консоль и удалил сервис (`kubectl delete svc`) — ArgoCD увидит «отклонение» (Out of Sync) и **мгновенно создаст его заново**, вернув кластер к состоянию из Git.

---

![[Pasted image 20260328190530.png]]


#### Работа с argoCD: это создание [[./application|application]] - сущность для управления деплойментом приложения

---

#### Как должен выглядеть ArgoCD Git Repository


```bash
Git repository
│
├── HelmCharts
│	├── ChartTest1
│	│	├── Chart.yaml
│	│   ├── templates/
│	│	├── values_dev.yaml
│	│	├── values_prod.yaml
│	│	└── values.yaml
│	│
│	└── ChartTest2
│		├── Chart.yaml
│	    ├── templates/
│		├── values_dev.yaml
│		├── values_prod.yaml
│		└── values.yaml
│
├── demo-dev                  
│   ├── applications
│	│    ├── app1.yaml
│	│    └── app2.yaml
│   └── root.yaml
│
└── demo-prod                  
    ├── applications
	│    ├── app1.yaml
	│    └── app2.yaml
    └── root.yaml
```