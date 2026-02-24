# Найти и убить процесс
kill -9 $(ps aux | grep process | awk '{print $2}')

# Мониторинг лога в реальном времени
tail -f /var/log/syslog | grep ERROR

# Подсчет строк в файлах
find . -name "*.js" | xargs wc -l

# Скачать весь сайт
wget -r -l 10 -k -p http://example.com

# Бэкап MySQL баз
mysqldump --all-databases | gzip > backup.sql.gz

# Найти большие файлы
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null

# Мониторинг в реальном времени
watch -n 1 'ps aux | grep nginx'

# Сгенерировать пароль
openssl rand -base64 32

# Выполнить команду на всех серверах
for host in server{1..10}; do ssh $host "uptime"; done