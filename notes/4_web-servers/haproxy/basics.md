##### **HAProxy это**: Высокопроизводительный TCP/HTTP балансировщик и прокси. Он не умеет отдавать статику, но идеален для балансировки и обеспечения отказоустойчивости.

#### Архитектура: Есть фронтенд (где слушает), бэкенд (куда отправляет) и статистика.

#### Базовый конфиг (`/etc/haproxy/haproxy.cfg`):

```haproxy
global
	log /dev/log local0
	maxconn 4096
	user haproxy
	group haproxy
	
defaults
	log global
	mode http
	option httplog
	option dontlognull
	retries 3
	timeout connect 5000
	timeout client 50000
	timeout server 50000
	
frontend http-in
	bind *:80
	default_backend servers
	
backend servers
	balance roundrobin
	server web1 10.0.0.1:80 check
	server web2 10.0.0.2:80 check
```

#### **Режимы:**

- **mode http:** Работа с HTTP (заголовки, куки).

- **mode tcp:** Работа на уровне TCP (для баз данных, WebSockets).

- **Балансировка:** `balance roundrobin`, `balance leastconn`, `balance source` (аналог IP hash).

- **Health Checks:** `check` в конце строки сервера — HAProxy будет проверять, жив ли сервер, и не слать туда трафик, если он умер.

- **Статистика:** Можно включить веб-страницу для мониторинга.

```haproxy
listen stats
	bind *:8080
	stats enable
	stats uri /stats
	stats refresh 10s
```