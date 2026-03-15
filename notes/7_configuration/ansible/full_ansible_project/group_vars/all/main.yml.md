```yml
ansible_ssh_private_key_file: "~/.ssh/id_rsa" 
ansible_user: "{{SERVER_U}}"
ansible_sudo_pass: "{{SERVER_P}}"
ansible_python_interpreter: "/usr/bin/python3"
host_key_checking: false

app_dir: /rent

domain: kycaka.com

certbot_image: certbot/certbot:latest
mysql_image: mysql:8.0.40
nginx_image: nginx:1.27-alpine


db_name: "{{DB_N}}"
db_user: "{{DB_U}}"
db_password: "{{DB_P}}"
mysql_root_password: "{{DB_ROOT_P}}"
jwt_access_key: "{{ACCESS}}"
jwt_refresh_key: "{{REFRESH}}"
docker_username: "{{REGISTRY_U}}"
docker_password: "{{REGISTRY_P}}"
image_name: "{{REGISTRY_IMG}}"
admin_email: "{{ADMIN_E}}"
gmail_user: "{{EMAIL_U}}"
gmail_pass: "{{EMAIL_P}}"
container_name: "app"
```