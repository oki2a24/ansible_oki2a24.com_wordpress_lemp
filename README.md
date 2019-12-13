# ansible_oki2a24.com_wordpress_lemp
oki2a24.com 用の WordPress の LEMP サーバを構築する Ansible プレイブックです。

# 疎通確認
```bash
ansible-playbook -i hosts test.yml
ansible-playbook -i hosts site.yml -v --tags common
```

---

今動いている設定。これを使いたい。

```conf:default.conf
[centos@ip-172-26-1-202 ~]$ sudo cat /etc/nginx/conf.d/default.conf
root   /srv/wordpress/;
index  index.php index.html index.htm;

proxy_cache_path  /var/cache/nginx keys_zone=czone:32m levels=1:2 inactive=3d max_size=256m;

server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;
    server_name oki2a24.com;
    return 301 https://$host$request_uri;
}

server {
#    listen 80 default_server;
#    listen [::]:80 default_server ipv6only=on;
#    server_name oki2a24.com;
    # For https
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server ipv6only=on;
    ssl_certificate /etc/letsencrypt/live/oki2a24.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/oki2a24.com/privkey.pem;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'kEECDH+ECDSA+AES128 kEECDH+ECDSA+AES256 kEECDH+AES128 kEECDH+AES256 kEDH+AES128 kEDH+AES256 DES-CBC3-SHA +SHA !aNULL !eNULL !LOW !kECDH !DSS !MD5 !EXP !PSK !SRP !CAMELLIA !SEED';
    add_header Strict-Transport-Security 'max-age=31536000; includeSubDomains;';
    ssl_stapling on;
    ssl_session_cache builtin:1000 shared:SSL:10m;

    proxy_cache czone;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_cache_valid 200 404 1d;

    location ~ /\. {deny all; access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }
    location = /favicon.ico { access_log off; log_not_found off; }
    location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
        log_not_found off;
        proxy_pass http://unix:/var/run/nginx.sock;
    }

    set $do_not_cache 0;

    if ($request_method != "GET") {
        set $do_not_cache 1;
    }

    if ($uri ~* "\.php$") {
        set $do_not_cache 1;
    }

    if ($http_cookie ~ ^.*(comment_author_|wordpress_logged_in|wp-postpass_).*$) {
        set $do_not_cache 1;
    }

    location / {
        proxy_no_cache $do_not_cache;
        proxy_cache_bypass $do_not_cache;
        proxy_cache_key "$scheme://$host$request_uri";
        proxy_pass http://unix:/var/run/nginx.sock;
    }

}

server {
    listen unix:/var/run/nginx.sock;
    try_files $uri $uri/ /index.php?q=$uri&$args;

    location ~ \.php$ {
        fastcgi_pass   unix:/var/run/php-fpm/php-fpm.sock;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
        fastcgi_pass_header "X-Accel-Redirect";
        fastcgi_pass_header "X-Accel-Buffering";
        fastcgi_pass_header "X-Accel-Charset";
        fastcgi_pass_header "X-Accel-Expires";
        fastcgi_pass_header "X-Accel-Limit-Rate";
    }

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
[centos@ip-172-26-1-202 ~]$
```
