
| Команда	 | Описание	                  | Пример                         |
|------------|----------------------------|--------------------------------|
| ip	     | Настройка сети	          | ip addr show (IP адреса)       |
| ifconfig	 | Устаревшая	              | ifconfig -a                    |
| ping	     | Проверка доступности	      | ping -c 4 google.com           |
| netstat	 | Сетевые соединения	      | netstat -tulpn                 |
| ss	     | Современная замена netstat |	ss -tulpn                      |
| curl	     | HTTP запросы	              | curl -I http://example.com     |
| wget	     | Скачивание	              | wget https://example.com/file  | 
| traceroute | Маршрут	                  | traceroute google.com          |
| nslookup	 | DNS запросы	              | nslookup google.com            |
| dig	     | Расширенный DNS	          | dig google.com                 |
| nc	     | Сетевой Swiss Army Knife	  | nc -zv host port               |
| iptables	 | Файрвол	                  | iptables -L -n                 |
| ufw	     | Простой файрвол	          | ufw status                     |


### SSH:

ssh user@host                    # Подключиться
ssh -p 2222 user@host            # На другой порт
ssh -i key.pem user@host         # С ключом
ssh-copy-id user@host             # Скопировать ключ

# Туннелирование
ssh -L 8080:localhost:80 user@host  # Проброс порта
ssh -D 1080 user@host                # SOCKS прокси

# Копирование через SSH
scp file user@host:/path/        # Скопировать файл
scp -r dir user@host:/path/      # Скопировать папку
rsync -av dir/ user@host:/path/  # Синхронизация