###### **Apache это:** Самый старый и проверенный веб-сервер. До сих пор рулит на shared-хостинге из-за `.htaccess`.

#### **Основные концепции:**

- **Модули:** Почти всё в Apache — модули. Их можно подключать и отключать (`a2enmod`, `a2dismod`).

- **.htaccess:** Позволяет менять конфигурацию прямо из папки с сайтом, без доступа к основному конфигу. Удобно для пользователей, но замедляет работу (Apache каждый раз сканирует папки в поисках этих файлов).

#### **Установка на Ubuntu:**

```bash
sudo apt update
sudo apt install apache2 -y
```

#### **Конфиги:**

- `/etc/apache2/apache2.conf` — главный конфиг.

- `/etc/apache2/sites-available/` — доступные сайты.

- `/etc/apache2/sites-enabled/` — включенные сайты (симлинки).

- `/etc/apache2/ports.conf` — какие порты слушать.

#### **Virtual Host (как server block в nginx):**

```apache
<VirtualHost *:80>
ServerName example.com
ServerAlias www.example.com
DocumentRoot /var/www/example
ErrorLog ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- **Команды управления:**

- `sudo systemctl start apache2`

- `sudo a2ensite example.com.conf` (включить сайт)

- `sudo a2dissite example.com.conf` (выключить)

- `sudo apache2ctl configtest` (проверка конфига)