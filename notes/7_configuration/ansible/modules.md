
## Модули - это маленькие «рабочие лошадки» Ansible, отдельные программы, которые выполняют конкретную задачу на целевом узле. Их существует очень много > 3000. 
## Категории основных модулей Ansible

Модули можно разделить на несколько основных категорий:

| Категория                 | Назначение                      | Примеры модулей                          |
| ------------------------- | ------------------------------- | ---------------------------------------- |
| **Управление пакетами**   | Установка/удаление ПО           | `apt`, `yum`, `pip`, `npm`               |
| **Работа с файлами**      | Копирование, шаблоны, права     | `copy`, `template`, `file`, `lineinfile` |
| **Выполнение команд**     | Запуск скриптов и команд        | `command`, `shell`, `script`, `raw`      |
| **Управление сервисами**  | Запуск/остановка служб          | `service`, `systemd`                     |
| **Пользователи и группы** | Создание/удаление пользователей | `user`, `group`                          |
| **Сеть и безопасность**   | Настройка сети, фаерволы        | `firewalld`, `iptables`, `uri`           |
| **Работа с Git**          | Клонирование репозиториев       | `git`                                    |
| **Сбор информации**       | Получение данных о системе      | `setup`, `stat`                          |


### Синтаксис основных команд:

- ##### **command** - Выполнение команд (без shell).

**Для чего:** Самый безопасный способ выполнить простую команду на удаленном сервере. Не использует shell, поэтому не обрабатывает `|`, `>`, `<`, `&`. В случаях когда нужны возможности shell: пайпы, перенаправление, переменные окружения нужно использовать shell (менее безопасный) 

```yml
- name: Проверить версию ядра
  command: uname -r
  register: kernel_version

- name: Создать директорию (без shell)
  command: mkdir -p /opt/myapp
  args:
    creates: /opt/myapp  # Не выполнять, если директория уже есть


# Ad-hoc команда 
ansible all -m command -a 'uptime'

```


- ##### **copy** - Копирование файлов с управляющей машины на управляемые хосты

```yml
- name: Скопировать файл конфигурации
  copy:
    src: /home/user/nginx.conf
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
	directory_mode: no # для директорий
	
	
# Ad-hoc команда

ansible all -m copy -a 'src=/etc/hosts dest=/tmp/hosts backup=yes'

```


- ##### **file** - Управление файлами и директориями.

```yml
- name: Создать директорию
  file:
    path: /opt/myapp
    state: directory
    owner: appuser
    group: appgroup
    mode: '0755'

- name: Создать пустой файл
  file:
    path: /var/log/myapp.log
    state: touch
    owner: appuser

- name: Создать символическую ссылку
  file:
    src: /opt/myapp/current
    dest: /opt/myapp/releases/v1.0.0
    state: link

- name: Удалить файл/директорию
  file:
    path: /tmp/oldfile.txt
    state: absent

- name: Рекурсивно изменить права
  file:
    path: /opt/myapp
    state: directory
    recurse: yes
    owner: appuser

   
# Ad-hoc команда
ansible all -m file -a 'path=/tmp/test state=touch'

```


- ##### **template** - Шаблоны файлов (Jinja2): копирует файлы с подстановкой переменных через шаблонизатор Jinja2.

```yml
- name: Сгенерировать конфиг nginx из шаблона
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/sites-available/{{ domain }}.conf
    owner: root
    mode: '0644'
  vars:
    domain: example.com
    port: 8080



# Пример шаблона nginx.conf.j2:

server {
    listen {{ port }};
    server_name {{ domain }};
    root /var/www/{{ domain }};
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```


- ##### **service / systemd** - Управление сервисами: запуск, остановка, перезапуск.

```yml
- name: Запустить nginx и добавить в автозагрузку
  service:
    name: nginx
    state: started
    enabled: yes

- name: Перезапустить MySQL
  systemd:
    name: mysql
    state: restarted
    daemon_reload: yes

- name: Остановить и отключить сервис
  service:
    name: apache2
    state: stopped
    enabled: no


# Ad-hoc команда
ansible all -m service -a 'name=nginx state=restarted'

```

- ##### **apt / yum / package** - Управление пакетами: Установка, удаление, обновление.

```yml
# Debian/Ubuntu:

- name: Установить несколько пакетов
  apt:
    name:
      - nginx
      - git
      - curl
    state: present
    update_cache: yes

- name: Установить конкретную версию
  apt:
    name: docker-ce=5:20.10.24-3~ubuntu-focal
    state: present

- name: Удалить пакет
  apt:
    name: apache2
    state: absent
    purge: yes


# RHEL/CentOs:

- name: Установить пакеты
  yum:
    name: httpd
    state: latest
    
    
# Универсальный модуль package:
    
- name: Установить git (работает на любой ОС)
  package:
    name: git
    state: present


# Ad-hoc команда
ansible all -m apt -a 'name=nginx state=present update_cache=yes' --become # sudo

```


- ##### **user / group** - Управление пользователями: создание, удаление, модификация групп и пользователей.

```yml
- name: Создать группу
  group:
    name: appgroup
    gid: 2000
    state: present

- name: Создать пользователя
  user:
    name: appuser
    uid: 1001
    group: appgroup
    groups: docker,sudo
    home: /home/appuser
    shell: /bin/bash
    password: "{{ hashed_password }}"
    state: present
    append: yes  # Добавить к группам, не удаляя из других

- name: Добавить SSH ключ пользователю
  authorized_key:
    user: appuser
    state: present
    key: "{{ lookup('file', '/home/user/.ssh/id_rsa.pub') }}"

- name: Удалить пользователя
  user:
    name: olduser
    state: absent
    remove: yes
```


- ##### **git** - Работа с Git репозиториями: клонирование и обновление

```yml
- name: Склонировать репозиторий
  git:
    repo: https://github.com/user/myapp.git
    dest: /opt/myapp
    version: main  # ветка, тег или коммит
    update: yes
    force: yes
    depth: 1  # shallow clone (быстрее)
    accept_hostkey: yes

- name: Склонировать приватный репозиторий
  git:
    repo: git@github.com:user/private-repo.git
    dest: /opt/private
    key_file: /home/appuser/.ssh/id_rsa
    accept_hostkey: yes


# Ad-hoc команда
ansible all -m git -a 'repo=https://github.com/ansible/ansible.git dest=/opt/ansible'
```


- ##### **setup** - Сбор информации (facts)

```yml
- name: Собрать информацию о системе
  setup:
    filter: "ansible_*"  # Можно фильтровать

- name: Использовать facts в плейбуке
  debug:
    msg: "Система: {{ ansible_os_family }}, IP: {{ ansible_default_ipv4.address }}"
    
    
# Ad-hoc команда
ansible all -m setup | grep ansible_os_family


# Полезные facts:
ansible_os_family: "Debian"  # или "RedHat"
ansible_distribution: "Ubuntu"
ansible_distribution_version: "20.04"
ansible_processor_cores: 4
ansible_memory_mb.real.total: 8192
ansible_default_ipv4.address: "192.168.1.10"
```


## Разные другие полезные модули:

```yml
#### Для работы с базами данных

- name: Создать базу MySQL
  mysql_db:
    name: myapp
    state: present

- name: Создать пользователя MySQL
  mysql_user:
    name: appuser
    password: "{{ db_password }}"
    priv: 'myapp.*:ALL'
    state: present



#### Для работы с Docker

- name: Запустить Docker контейнер
  docker_container:
    name: myapp
    image: "{{ image_name }}"
    state: started
    ports:
      - "5000:5000"
    env:
      NODE_ENV: production
      


### Для работы с cron (планировщик задач)      
      
- name: Добавить задачу в cron
  cron:
    name: "backup database"
    minute: "0"
    hour: "2"
    job: "/usr/bin/backup.sh"
    user: root



#### Для HTTP запросов
    
- name: Проверить доступность сайта
  uri:
    url: https://example.com
    method: GET
    status_code: 200
    return_content: yes
  register: webpage

- name: Отправить POST запрос к API
  uri:
    url: https://api.example.com/users
    method: POST
    body: "name=John&age=30"
    headers:
      Content-Type: "application/x-www-form-urlencoded"
    status_code: 201
    
    
    
##### Для отладки    
    
- name: Вывести переменную
  debug:
    msg: "Значение переменной: {{ my_var }}"
    verbosity: 1

- name: Показать все переменные
  debug:
    var: ansible_facts    

```