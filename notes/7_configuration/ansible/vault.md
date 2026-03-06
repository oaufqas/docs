## Ansible Vault - Шифрует конфиденциальные данные (пароли, ключи) в плейбуках или переменных.

##### Создание зашифрованного файла:

```bash
ansible-vault create secrets.yml
```

##### Использование в плейбуке:

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