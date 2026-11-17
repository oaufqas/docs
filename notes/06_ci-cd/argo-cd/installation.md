### Для использования gitops инструмента argocd, его нужно установить в кластер c kubectl apply или helm chart:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side
```

После установки, чтобы зайти в графическую оболочку, нужно узнать пароль, который генерируется при установке (можно перегенерировать)

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Например 6IBtOX-s7phy9f

# Порт форвард для доступа графической оболочки
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address 0.0.0.0
```

Графическая оболочка будет доступна по `localhost:8080`, логин `admin` 