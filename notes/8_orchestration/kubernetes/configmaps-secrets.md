### Это инструменты для отделения конфигурации от кода.

## 1. ConfigMap (Публичные настройки)

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


## 2. Secret (Конфиденциальные данные)

Используется для паролей, токенов, ключей SSH или SSL-сертификатов.

- **Что храним:** `DB_PASSWORD`, `STRIPE_API_KEY`, `tls.crt`.
- **Безопасность:** Внутри Kubernetes они хранятся в формате **Base64** (это НЕ шифрование, это просто кодировка). В `etcd` они могут быть зашифрованы, если админ это настроил.
- **Нюанс:** При выводе `kubectl get secret ... -o yaml` ты увидишь абракадабру (Base64). Чтобы прочитать, нужно декодировать: `echo "ZGF0YQ==" | base64 -d`.

Пример манифеста:

```yaml
# echo -n 'adminuser' | base64
# echo -n 'password' | base64
apiVersion: v1
kind: Secret
metadata: 
  name: secret-data
type: Opaque
data:
  username: FDiuywND2bnk
  password: PaG2gapigb3219==
  
  
--- # Использование в deployment
spec:
  containers:
  - name: js
    image: rvlxx/js:1.0
    ports: 
    - containerPort: 8000
    envFrom:
    - secretRef:
        name: secret-data
        



--- # 2 вариант
  
apiVersion: v1
kind: Secret
metadata:
  name: secret-stringdata
type: Opaque
stringData: # Автоматически закодирует в base64
  username: admin
  password: nopass

  
--- # Использование в deployment
spec:
  containers:
  - name: js
    image: rvlxx/js:1.0
    ports: 
    - containerPort: 8000
    env:
    - name: SECRET_USERNAME
      valueForm: 
        secretKeyRef:
          name: secret-stringdata
          key: username
          
	- name: SECRET_PASSWORD
      valueForm: 
        secretKeyRef:
          name: secret-stringdata
          key: password
          
          
--- # 3 Registy secret
apiVersion: v1
kind: Secret
metadata:
  name: registry-secret
type: kubernetes.io/dockerconfigjson
data:
  # Это base64 от конфига Docker
  .dockerconfigjson: <закодированная строка>

```


Команды для работы с секретами:

```bash
echo -n '{"auths":{"https://index.docker.io/v1/":{"username":"username","password":"pass","auth":"'$(echo -n "username:pass" | base64)'"}}}' | base64 -w 0
```

```bash
# Создать секрет из файлов командой:
kubectl create secret generic db-user-pass \
--from-file=./username.txt \
--from-file=./password.txt

# Создать секрет напрямую:
kubectl create secret generic db-user-pass \
--from-literal=username=devuser \
--from-literal=password='J3afHGba4tgadf@1!'

# Создать секрет из .env файла !!
kubectl create secret generic app-secret --from-env-file=.env
```

Создать секрет для pull из docker registry

```bash
kubectl create secret docker-registry <имя-секрета> \
  --docker-server=<url-реестра> \
  --docker-username=<логин> \
  --docker-password=<пароль-или-токен> \
  --docker-email=<ваш-email>
```

Получение данных

```bash
# Полное инфо сущности
kubectl get secret db-user-pass -o yaml

# Извлечение и рассшифровка пароля
kubectl get secret db-user-pass -o jsonpath='{.data.password}' | base64 -d 
```