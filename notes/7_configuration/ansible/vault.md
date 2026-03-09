## Ansible Vault - это встроенный инструмент для шифрования данных, который позволяет безопасно хранить пароли, ключи API, сертификаты и другие секреты в плейбуках и ролях.


#### Основные команды:

```bash
# Создание зашифрованного файла
ansible-vault create <filename>

# Редактировать содержимое уже зашифрованного файла
ansible-vault edit <filename>

# Зашифровать существующий текстовый файл
ansible-vault encrypt <filename>

# Навсегда расшифровать файл (вернуть в открытый вид).
ansible-vault decrypt <filename>

# Просмотреть содержимое без открытия в редакторе
ansible-vault view <filename>
```


Как использовать в Playbook

При запуске плейбука, содержащего зашифрованные данные, Ansible нужно предоставить пароль. Сделать это можно двумя способами:
1. **Интерактивно**: добавить флаг `--ask-vault-pass` при запуске `ansible-playbook`.
2. **Через файл**: указать путь к файлу с паролем через параметр `--vault-password-file` или переменную окружения `ANSIBLE_VAULT_PASSWORD_FILE`.


```yml
- name: Deploy app
  hosts: all
  vars_files:
    - secrets.yml
  tasks:
    - name: DB password
      debug:
        msg: "Пароль: {{ db_password }}"

```