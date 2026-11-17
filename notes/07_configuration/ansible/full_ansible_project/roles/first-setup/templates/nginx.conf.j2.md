```conf
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;

events {
        worker_connections 768;
}

http {
        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;

        access_log /var/log/nginx/access.log;

        gzip on;

        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 6;
        gzip_buffers 16 8k;
        gzip_http_version 1.1;
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
        
        server {
                listen 80;
                listen [::]:80;
                server_name {{domain}} www.{{domain}};
                return 301 https://$server_name$request_uri;
        }

        server {
                listen 443 ssl http2;
                listen [::]:443 ssl http2;

                client_max_body_size 200M;
                client_body_timeout 100s;
                client_body_buffer_size 256k;

                server_name {{domain}} www.{{domain}};

                ssl_certificate /etc/letsencrypt/live/{{domain}}/fullchain.pem;
                ssl_certificate_key /etc/letsencrypt/live/{{domain}}/privkey.pem;
                
                ssl_session_cache shared:le_nginx_SSL:10m;
                ssl_session_timeout 1440m;
                ssl_session_tickets off;
                ssl_protocols TLSv1.2 TLSv1.3;
                ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384";
                ssl_prefer_server_ciphers off;

                location / {
                        proxy_pass http://app:5000;
                        proxy_http_version 1.1;
                        proxy_set_header Upgrade $http_upgrade;
                        proxy_set_header Connection 'upgrade';
                        proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                        proxy_set_header X-Forwarded-Proto $scheme;
                        proxy_cache_bypass $http_upgrade;

                        client_max_body_size 200M;
                        proxy_connect_timeout 100s;
                        proxy_send_timeout 100s;
                        proxy_read_timeout 100s;
                }
        }
}

```