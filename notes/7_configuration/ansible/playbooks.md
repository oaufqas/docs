## Playbooks - YAML-файлы, описывающие сценарии: какие задачи (tasks) выполнять на хостах из Inventory.

##### Например playbook.yml:

```yml
--- # Начало YAML файла плейбука
- name: Полная настройка веб-сервера # Отображаемое имя
  hosts: webservers # На каких хостах выполнять (inventory)
  become: yes # Использовать права sudo
  vars: # Словарь переменных
    app_port: 8080
    domain: example.com
    packages:
      - nginx
      - git
      - curl

  tasks: # Список задач
    - name: Установка пакетов # Описание задачи
      apt: # Модуль apt 
        name: "{{ packages }}" # Подставление значения переменной
        update_cache: yes # Обновлять кэш пакетов
        state: present # Состояние: установлен
      tags: install # Метка для выборочного запуска

    - name: Создать пользователя приложения # Новая задача
      user: # Модуль user
        name: appuser
        home: /opt/app
        shell: /bin/bash
      tags: user # Метка для выборочного запуска
      
    - name: Клонировать репозиторий # Новая задача
      git: # Модуль git 
        repo: https://github.com/user/myapp.git
        dest: /opt/app
        version: main
      tags: deploy # Метка для выборочного запуска
      register: git_result # Сохранить результат в переменную

    - name: Сгенерировать конфиг nginx
      template: # Модуль template (Jinja2)
        src: nginx.conf.j2 # Файл шаблона локально
        dest: /etc/nginx/sites-available/{{ domain }}
      notify: restart nginx # Вызов обработчика при изменении
      tags: configure # Метка для выборочного запуска

    - name: Включить сайт
      file: # Модуль file
        src: /etc/nginx/sites-available/{{ domain }}
        dest: /etc/nginx/sites-enabled/{{ domain }}
        state: link # Создать символическую ссылку
      notify: restart nginx # Вызов обработчика при изменении
      tags: configure # Метка для выборочного запуска

    - name: Запустить приложение
      shell: | # Модуль shell (многострочная строка)
        cd /opt/app
        npm install
        npm run prod
      when: git_result.changed # Условие: выполнить только если изменился репозиторий
      tags: start # Метка для выборочного запуска

  handlers: # Список обработчиков (запускаются по notify)
    - name: restart nginx
      service: # Модуль service
        name: nginx
        state: restarted
```


### Основные директивы (Ключевые элементы):

- **hosts** - Кого настраиваем
```yml
- hosts: all                    # Все серверы из инвентаря
- hosts: webservers             # Только группа webservers
- hosts: "{{ target_hosts }}"   # Из переменной
- hosts: 192.168.1.10           # Конкретный IP
- hosts: webservers:databases   # Несколько групп
```
- **name** - Описание задачи
```yml
tasks:
  - name: Установить nginx на веб-сервер
    apt:
      name: nginx
      state: present

  - name: Скопировать конфиг приложения
    copy:
      src: app.conf
      dest: /etc/app/
```
- **vars** - Переменные
```yml
- hosts: all
  vars:
    app_port: 8080
    app_user: myapp
    packages:
      - git
      - curl
      - vim
  tasks:
    - name: Использовать переменную
      debug:
        msg: "Приложение будет на порту {{ app_port }}"
    
    - name: Установить пакеты
      apt:
        name: "{{ packages }}"
        state: present
```
- **vars_files** - Переменные из файлов
```yml
- hosts: all
  vars_files:
    - vars/common.yml
    - "vars/{{ env }}.yml"  # Динамический файл
  tasks:
    - name: Использовать переменные из файла
      debug:
        msg: "DB host: {{ database_host }}"
```
- **become** - Повышение привилегий
```yml
- hosts: all
  become: yes                # Все задачи от root
  become_user: postgres       # От конкретного пользователя
  become_method: sudo         # Способ (sudo/su)
  tasks:
    - name: Эта задача от root
      command: whoami
      register: result
    - debug:
        msg: "Я {{ result.stdout }}"  # Выведет "root"

    - name: Эта задача от postgres (переопределяем)
      command: whoami
      become_user: postgres
      register: result
    - debug:
        msg: "Я {{ result.stdout }}"  # Выведет "postgres"
```


## Управление выполнением 


- **when** - Условное выполнение
```yml
tasks:
  - name: Установить только на Debian
    apt:
      name: nginx
    when: ansible_os_family == "Debian"

  - name: Установить только на CentOS
    yum:
      name: nginx
    when: ansible_os_family == "RedHat"

  - name: Выполнить если переменная определена
    debug:
      msg: "{{ database_host }}"
    when: database_host is defined

  - name: Сложное условие
    command: /usr/bin/upgrade
    when:
      - ansible_distribution == "Ubuntu"
      - ansible_distribution_version == "20.04"
      - force_upgrade | default(False)
```
- **register** - Сохранение результата
```yml
tasks:
  - name: Выполнить команду
    command: /usr/bin/myapp --version
    register: version_output

  - name: Показать результат
    debug:
      msg: "Версия: {{ version_output.stdout }}"

  - name: Использовать в условии
    command: /usr/bin/update
    when: version_output.rc == 0  # exit code 0 - успех

  - name: Работа с JSON результатом
    uri:
      url: https://api.example.com/status
    register: api_result

  - name: Проверить статус из JSON
    debug:
      msg: "API статус: {{ api_result.json.status }}"
```
- **ignore_errors** - Игнорирование ошибок
```yml
tasks:
  - name: Попробовать удалить (если нет - не страшно)
    file:
      path: /tmp/oldfile
      state: absent
    ignore_errors: yes

  - name: Выполнить даже если предыдущее упало
    command: echo "Все равно выполнится"
```
- **failed_when** - Свои условия ошибки
```yml
tasks:
  - name: Проверить лог
    command: grep "ERROR" /var/log/app.log
    register: grep_result
    failed_when:
      - grep_result.rc == 0  # Если найдена ошибка - считаем job проваленным
      - '"CRITICAL" in grep_result.stdout'
```
- **changed_when** - Свои условия изменений
```yml
tasks:
  - name: Запустить скрипт
    shell: /usr/bin/update-db
    register: script_result
    changed_when:
      - '"updated" in script_result.stdout'
      - script_result.rc == 0
```

## Итерации и циклы


- **loop** - Перебор списка (**with_items** - старый синтаксис)
```yml
tasks:
  - name: Установить несколько пакетов
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - git
      - curl

  - name: Создать несколько пользователей
    user:
      name: "{{ item.name }}"
      groups: "{{ item.groups }}"
      shell: /bin/bash
    loop:
      - { name: 'alice', groups: 'users' }
      - { name: 'bob', groups: 'sudo' }
      - { name: 'charlie', groups: 'docker' }
```
- **with_dict** - Перебор словаря
```yml
tasks:
  - name: Создать пользователей из словаря
    user:
      name: "{{ item.key }}"
      uid: "{{ item.value.uid }}"
      groups: "{{ item.value.groups }}"
    with_dict:
      alice:
        uid: 1001
        groups: users
      bob:
        uid: 1002
        groups: sudo
```


## Хранение данных


- **vars_prompt** - Запрос ввода
```yml
- hosts: all
  vars_prompt:
    - name: database_password
      prompt: "Введите пароль для БД"
      private: yes           # Не показывать при вводе
      confirm: yes           # Подтверждение
      default: "admin123"    # Значение по умолчанию
  tasks:
    - name: Использовать пароль
      debug:
        msg: "Пароль: {{ database_password }}"
```
- **set_fact** - Создание переменных
```yml
tasks:
  - name: Получить дату
    command: date +%Y%m%d
    register: date_result

  - name: Сохранить как факт
    set_fact:
      backup_date: "{{ date_result.stdout }}"
      backup_path: "/backups/{{ date_result.stdout }}"

  - name: Использовать созданные факты
    file:
      path: "{{ backup_path }}"
      state: directory
```


## Обработчики (handlers)


- **handlers** - Задачи по уведомлению
```yml
tasks:
  - name: Изменить конфиг nginx
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: restart nginx  # Отправить уведомление

  - name: Изменить конфиг приложения
    template:
      src: app.conf.j2
      dest: /etc/app/config.conf
    notify: restart app

handlers:
  - name: restart nginx  # Запустится только если была notify
    service:
      name: nginx
      state: restarted

  - name: restart app
    systemd:
      name: myapp
      state: restarted
```


## Подключение файлов


- **include_tasks** / **import_tasks** - Подключение файла задач
```yml
tasks:
  - name: Включить файл с задачами
    include_tasks: install-packages.yml

  - name: Импортировать задачи
    import_tasks: configure-app.yml
    when: env == "production"
```
- include_vars - Подключение переменных
```yml
tasks:
  - name: Загрузить переменные для окружения
    include_vars: "vars/{{ env }}.yml"

  - name: Загрузить все .yml файлы из папки
    include_vars:
      dir: vars
      extensions:
        - yml
        - yaml
```


## Roles


- **roles** - Использование ролей
```yml
- hosts: webservers
  roles:
    - common           # Базовая роль
    - role: nginx      # С переменными
      vars:
        port: 8080
        workers: 4
    - role: postgresql
      when: install_db | default(False)
```
- **apply** - Применить роли с условиями
```yml
- hosts: app
  tasks:
    - name: Применить роль с условием
      import_role:
        name: security
      when: ansible_os_family == "Debian"
```


## Теги (tags)


- **tags** - Маркировка задач
```yml
tasks:
  - name: Установить пакеты
    apt:
      name: nginx
    tags:
      - install
      - nginx

  - name: Настроить конфиг
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    tags:
      - configure
      - nginx

  - name: Запустить сервис
    service:
      name: nginx
      state: started
    tags:
      - start
      - nginx
```


## Блоки и стратегии выаолнения 


- **block** - Группировка задач
```yml
tasks:
  - name: Блок установки и настройки
    block:
      - name: Установить пакеты
        apt:
          name: "{{ packages }}"

      - name: Настроить конфиг
        template:
          src: app.conf.j2
          dest: /etc/app.conf

      - name: Запустить сервис
        service:
          name: app
          state: started
    when: ansible_os_family == "Debian"
    become: yes
    ignore_errors: yes
```
- **rescue** и **always** (как try/catch)
```yml
tasks:
  - block:
      - name: Попытаться выполнить
        command: /usr/bin/might-fail
    rescue:
      - name: Если упало - выполняем это
        debug:
          msg: "Предыдущая задача упала, но мы живы"
    always:
      - name: Это выполнится всегда
        debug:
          msg: "Очистка..."
```