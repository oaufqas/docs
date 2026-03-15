**Server blocks это:** Виртуальные хосты. Позволяют держать на одном сервере несколько сайтов с разными доменами.

- **Пример:**

```nginx
server {
  listen 80;
  server_name site1.com www.site1.com;
  root /var/www/site1;
}
server {
  listen 80;
  server_name site2.com;
  root /var/www/site2;
}
```
- **Как создать:** Скопируй дефолтный конфиг, измени `server_name` и `root`, включи сайт (симлинк в `sites-enabled`).