## Inventory файл - это файл, содержащий список управляемых хостов, сгруппированных по функционалу.

Есть несколько форматов написания файла, например hosts.yml:

```yml
all: 
  hosts:
    web:
      ansible_host: 192.168.1.10
    db1:
      ansible_host: 192.168.1.11
      ansible_user: postgres
    db2: 
	  ansible_host: 192.168.1.12
	  ansible_user: redis
	  
  children:
    database:
      hosts:
        db1:
        db2:
        
  vars:
    ansible_ssh_private_key_file: ~/.ssh/ansible_key
```

Второй вариант hosts.ini:
```ini
[web]
192.168.1.10

[db]
192.168.1.11 ansible_user=postgres
192.168.1.12 ansible_user=redis
```


### Группы:

- По умолчанию всегда доступна группа `all`
- Можно создавать сколько угодно своих групп и обращаться к ним
- Если хосты в файле без группы, к ним можно обратиться с помощью `ungrouped`
### Параметры inventory:

 - `ansible_connection`: ssh/ winrm / local / docker
 -  `ansible_port`: 22 / 5986
 -  `ansible_user`: root
 -  `ansible_ssh_pass`,  `ansible_password`: password
 -  `ansible_ssh_private_key_file`: path/to/file
 -  `ansible_python_interpreter`: path/to/file