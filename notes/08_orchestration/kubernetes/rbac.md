### **ServiceAccount** (SA) -  Это встроенная сущность Kubernetes, которая выполняет роль «паспорта» для процессов внутри подов.

Как это работает (Цепочка доверия)

2. У пода есть **ServiceAccount**. Kubernetes автоматически подкладывает в этот под специальный JWT-токен.
3. Под (или Vault Agent) отправляет этот токен в Vault и говорит: «Я хочу зайти под ролью `app-role`».
4. Vault берет этот токен, идет к API Kubernetes и спрашивает: «Этот токен настоящий? К какому ServiceAccount он привязан?».
5. Если всё ок, Vault выдает поду свой временный **Vault Token** с правами, описанными в политике.

---

Создание аккаунтов в Kubernetes

```bash
# Создаем аккаунты
kubectl create serviceaccount server-sa
kubectl create serviceaccount db-sa
```

_Примечание: в манифесте Deployment/StatefulSet нужно будет указать `serviceAccountName: server-sa`_