### Это инструменты для отделения конфигурации от кода.

#### 1. ConfigMap (Публичные настройки)

![[Pasted image 20260401221352.png|416]]

Используется для хранения обычных текстовых данных: конфиг-файлов, переменных окружения, флагов запуска.

- **Что храним:** `API_URL`, `LOG_LEVEL`, `nginx.conf`, `index.html`.
- **Как выглядит:** Это просто словарь «ключ-значение».
- **Безопасность:** Данные хранятся в открытом виде. Любой, у кого есть доступ к кластеру, может их прочитать.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-conf
data:
  intnum: "7"
  default.conf: |
    server {
      listen 80 default_server;
      server_name _;
      
      default_type text/plain;
      
      location / {
        return 200 'MY-CONFIGMAP: $hostname\n';
      }
    }
    
    
--- # Подключение к Deployment
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
          - containerPort: 80
        env:
          - name: NUM
            valueFrom:
              configMapKeyRef:
                name: my-conf
                key: intnum
            
        volumeMounts:
          - name: config
            mountPath: /etc/nginx/conf.d/
            readOnly: true
            subPath: nginx.conf # Чтобы не удалять существующие файлы 
# При использовани subPath замена переменных в поде не будет динамеческой, нужно перезугружать под. При простом монтировании, все будет меняться динамически        
            
            
#        envFrom: - Импортировать все переменные
#        - prefix: CONFIG_DEV_
#          configMapRef:
#            name: my-conf
            
      volumes:
      - name: config
        configMap:
          name: my-conf
          items:
          - key: "default.conf"
            path: "default.conf"
```

Команды для работы с ConfigMap:

```bash
kubectl create configmap <name> --from-file=nginx.conf 
# Создание configmap с файлом из директории
--from-literal=5 # Со значением
--from-file=configs/ # Все файлы в директории
--from-env-file=env-file # К каждому ключу из .env файла (key=value)
```
#### 2. Secret (Конфиденциальные данные)

Используется для паролей, токенов, ключей SSH или SSL-сертификатов.

- **Что храним:** `DB_PASSWORD`, `STRIPE_API_KEY`, `tls.crt`.
- **Безопасность:** Внутри Kubernetes они хранятся в формате **Base64** (это НЕ шифрование, это просто кодировка). В `etcd` они могут быть зашифрованы, если админ это настроил.
- **Нюанс:** При выводе `kubectl get secret ... -o yaml` ты увидишь абракадабру (Base64). Чтобы прочитать, нужно декодировать: `echo "ZGF0YQ==" | base64 -d`.