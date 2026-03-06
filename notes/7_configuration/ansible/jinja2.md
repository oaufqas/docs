## Jinja2 (Шаблоны) - Позволяет создавать динамические конфигурационные файлы, подставляя переменные. Используются с модулем `template`

##### Пример шаблона (`roles/nginx/templates/nginx.conf.j2`):
```nginx
server {
    listen {{ http_port }};
    server_name {{ server_hostname }};
    root /var/www/html;
}

```