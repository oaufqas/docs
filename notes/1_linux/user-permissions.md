| Команда   | Описание             | Пример                  
|-----------|----------------------|-------------------------
| `chmod`   | Изменить права       | chmod 755 script.sh 
| `chown`   | Изменить владельца   | chown user:group file
| `umask`   | Маска по умолчанию   | umask 022           
| `ls -l`   | Посмотреть права     | ls -l                 
| `whoami`  | Текущий пользователь | whoami
| `id`	    | ID пользователя	   | id
| `useradd` | Создать пользователя | useradd -m -s /bin/bash user
| `usermod` | Изменить пользователя| usermod -aG docker user (Добавляет в группу)
| `userdel` | Удалить пользователя | userdel -r user (с домашней папкой)
| `passwd`  | Сменить пароль	   | passwd user
| `groupadd`| Создать группу	   | groupadd docker
| `groups`  | Группы пользователя  | groups user
| `sudo`	| Выполнить от root	   | sudo command
| `gpasswd` | Удалить из группы    | sudo gpasswd -d <имя_пользователя> <имя_группы>

**Права в цифрах:**
- 4 = read (r)
- 2 = write (w)
- 1 = execute (x)

**Примеры:**
- `chmod 755` = rwxr-xr-x (владелец всё, остальные чтение/выполнение)
- `chmod 644` = rw-r--r-- (владелец чтение/запись, остальные только чтение)
- `chown newuser newfile` – Меняет владельца newfile на newuser
- `chown newuser:newgroup newfile` – Изменяет владельца и группу-владельца для newfile на newuser и newgroup
- `chown -R newuser:newgroup newfolder` – Меняет владельца и группу-владельца каталога newfolder на newuser и newgroup
- `stat -c “%U %G” newfile` – отображает владельцев пользователей и групп newfile


cat /etc/group - Вывести все группы

Менять разрешения на использование sudo: etc/sudoers или sudo visudo