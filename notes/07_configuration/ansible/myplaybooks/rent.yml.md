```yaml
---

- name: Deploy rent ansible playbook
  hosts: all
  become: yes
  vars:
    app_dir: /rent
    mysql_image: mysql:8.0.40
    nginx_image: nginx:1.27-alpine
    certbot_image: certbot/certbot:latest  
  vars_files:
    - ./vars.yml
  tasks:

  # first setup server
  - name: Обновить кэш пакетов
    apt:
      update_cache: yes
      
  - name: Добавить GPG ключ Docker
    apt_key:
      url: https://download.docker.com/linux/ubuntu/gpg
      state: present  

  - name: Добавить репозиторий Docker
    apt_repository:
      repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
      state: present

  - name: Установить необходимые пакеты
    apt:
      name:
        - docker-ce
        - docker-ce-cli
        - containerd.io
        - docker-compose-plugin
        - python3-pip
        - python3-setuptools
      state: present
      update_cache: yes  

  - name: Установить Docker Python модуль
    apt:
      name:
        - python3-docker
      state: present

  - name: Запустить и добавить Docker в автозагрузку
    systemd:
      name: docker
      state: started
      enabled: yes  
  
  - name: Create project folder
    file:
      path: "{{app_dir}}/{{item}}"
      state: directory
      mode: '0755'
    loop:
      - "uploads/"
      - "nginx/"
      - "letsencrypt/"
      - "mysql/data/"

  - name: Copy template nginx
    template:
      src: ./nginx.conf.j2
      dest: "{{app_dir}}/nginx/nginx.conf"
      mode: '0644'

  - name: Copy init mysql
    copy:
      src: ./init.sql
      dest: "{{app_dir}}/mysql/init.sql"
      mode: '0644'

  - name: create docker network
    docker_network:
      name: app-network
      state: present



# renew ssl certificates


  - name: Проверить наличие сертификатов
    stat:
      path: "{{ app_dir }}/letsencrypt/live/{{ domain }}/fullchain.pem"
    register: cert_file  

  - name: Получить SSL сертификаты (standalone)
    docker_container:
      name: certbot-tmp
      image: "{{ certbot_image }}"
      state: started
      cleanup: yes
      ports:
        - "80:80"
      detach: no
      restart_policy: no
      volumes:
        - "{{ app_dir }}/letsencrypt:/etc/letsencrypt"
      command: "certonly --standalone --email {{ admin_email }} --agree-tos --no-eff-email -d {{ domain }}"
    when: not cert_file.stat.exists
    register: certbot_result

   - name: Перезапустить nginx (если сертификаты новые)
     docker_container:
       name: nginx
       restart: yes
     when: certbot_result.changed
  
  
  
  # start app containers
  

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
 
  - name: Логин в registry
    docker_login:
      username: "{{ docker_username }}"
      password: "{{ docker_password }}"
    no_log: true

  - name: Скачать новый образ
    docker_image:
      name: "{{image_name}}"
      source: pull
      force_source: yes
    register: pull_result

  - name: Остановить старый контейнер приложения
    docker_container:
      name: app
      state: absent
    ignore_errors: yes
  
  - name: Запустить с новым образом
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

  - name: Запустить Nginx контейнер
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
  

  - name: Проверить здоровье
    uri:
      url: "https://{{ domain }}/health"
      method: GET
      status_code: 200
    register: health
    retries: 5
    delay: 10
    until: health.status == 200  

  - name: Очистить старые образы
    docker_prune:
      images: yes
      images_filters:
        dangling: false
```