```bash

Debian/Ubuntu (apt)
apt update              # Обновить список пакетов
apt upgrade             # Обновить все пакеты
apt install package     # Установить пакет
apt remove package      # Удалить пакет
apt search term         # Поиск пакета
apt show package        # Информация о пакете


RHEL/CentOS (yum/dnf)

yum update              # Обновить все
yum install package     # Установить
yum remove package      # Удалить
yum search term         # Поиск



Alpine (apk)
bash
apk update              # Обновить список
apk add package         # Установить
apk del package         # Удалить
apk search term         # Поиск


rpm -i pkg_program.rpm – Устанавливает пакет rpm (CentOS, RHEL…)
rpm -e pkg_name – Удаляет пакет rpm (CentOS, RHEL…)
dnf install pkg_name – Устанавливает пакет с помощью dnf из репозитория. Ранее использовался YUM, но недавно YUM был заменен на DNF. (CentOS, RHEL…)

dpkg -i pkg_name – Установка из deb-пакета (Debian, Ubuntu, Mint…)
dpkg -r pkg_name – Удаляет deb-пакет (Debian, Ubuntu, Mint…)
```