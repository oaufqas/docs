```yml
---

- name: Start mysql container
  docker_container:
    name: mysql
    image: "{{mysql_image}}"
    state: started
    restart_policy: unless-stopped
    networks: 
      - name: app-network
    volumes:
      - "{{app_dir}}/mysql/data:/var/lib/mysql"
      - "{{app_dir}}/mysql/init.sql:/docker-entrypoint-initdb.d/init.sql"
    env:
      MYSQL_ROOT_PASSWORD: "{{ mysql_root_password }}"
      MYSQL_DATABASE: "{{ db_name }}"
      MYSQL_USER: "{{ db_user }}"
      MYSQL_PASSWORD: "{{ db_password }}"

    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
  register: mysql_result



- name: login in registry
  docker_login:
    username: "{{ docker_username }}"
    password: "{{ docker_password }}"
  no_log: true

- name: download new image
  docker_image:
    name: "{{image_name}}"
    source: pull
    force_source: yes
  register: pull_result

- name: stop the old application container
  docker_container:
    name: app
    state: absent
  ignore_errors: yes

- name: run app with a new image
  docker_container:
    name: "{{container_name}}"
    image: "{{image_name}}"
    state: started
    restart_policy: unless-stopped
    env:
      PORT: "5000"
      HOST: "0.0.0.0"
      DB_HOST: "mysql"
      DB_PORT: "3306"
      DB_NAME: "{{ db_name }}"
      DB_USER: "{{ db_user }}"
      DB_PASSWORD: "{{ db_password }}"
      JWT_ACCESS_KEY: "{{ jwt_access_key }}"
      JWT_REFRESH_KEY: "{{ jwt_refresh_key }}"
      ADMIN_EMAIL: "{{ admin_email }}"
      MYSQL_ROOT_PASSWORD: "{{ mysql_root_password }}"
      PROD: "false"
      GMAIL_USER: "{{ gmail_user }}"
      GMAIL_PASS: "{{ gmail_pass }}"
      API_URL: "https://{{ domain }}"
      CLIENT_URL: "https://{{ domain }}"
    networks:
      - name: app-network
    ports:
      - "465:465"
    volumes:
      - "{{ app_dir }}/uploads:/app/uploads"
  register: run_result



- name: run nginx container
  docker_container:
    name: nginx
    image: "{{ nginx_image }}"
    state: started
    restart_policy: unless-stopped
    networks:
      - name: app-network
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "{{ app_dir }}/nginx/nginx.conf:/etc/nginx/nginx.conf:ro"
      - "{{ app_dir }}/letsencrypt:/etc/letsencrypt:ro"
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3


- name: healthcheck
  uri:
    url: "https://{{ domain }}/health"
    method: GET
    status_code: 200
  register: health
  retries: 5
  delay: 10
  until: health.status == 200

- name: clear old images
  docker_prune:
    images: yes
    images_filters:
      dangling: false
```