### **На Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```
### **На CentOS/RHEL:**

```bash
sudo yum install epel-release -y
sudo yum install nginx -y
sudo systemctl start nginx
```

- **Проверка:** `nginx -t` (тест конфига), `systemctl status nginx`, `curl localhost`.

### В Docker container

```yml
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    networks:
      - app-network
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
```