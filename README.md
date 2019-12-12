# ansible_oki2a24.com_wordpress_lemp
oki2a24.com 用の WordPress の LEMP サーバを構築する Ansible プレイブックです。

## yum コマンドが失敗する
- [CentOSのrpmdbが壊れてyumコマンドが失敗する場合の修正方法](https://kaworu.jpn.org/kaworu/2015-11-14-1.php)

```bash
sudo rm -f /var/lib/rpm/__db*
sudo yum clean all
```

# 疎通確認
```bash
ansible-playbook -i hosts test.yml
ansible-playbook -i hosts site.yml -v --tags common
```

## WordPress に必要な PHP 拡張モジュール
- [System Packages Server Environment – Make WordPress Hosting](https://make.wordpress.org/hosting/handbook/handbook/server-environment/#system-packages)
- [PHP Extensions Server Environment – Make WordPress Hosting](https://make.wordpress.org/hosting/handbook/handbook/server-environment/#php-extensions)

```yml
# https://make.wordpress.org/hosting/handbook/handbook/server-environment/#system-packages
- name: Install system packages for WordPress
  yum:
    name:
      - ImageMagick6
      - ghostscript
      - ghostscript
      - libsodium
    enablerepo: epel,remi
    state: present

# https://make.wordpress.org/hosting/handbook/handbook/server-environment/#php-extensions
- name: PHP extensions for WordPress
  yum:
    name:
      - php-curl
      - php-dom
      - php-exif
      - php-fileinfo
      - php-hash
      - php-json
      - php-mbstring
      - php-mysqli
      - php-sodium
      - php-openssl
      - php-pcre
      - php-imagick
      - php-xml
      - php-zip
    enablerepo: epel,remi-php74
    state: latest
```

## WordPress ディレクトリに SELinux 設定を行う
- [Centos7.4でSELinuxを有効化し運用する](https://gtrt7.com/blog/linux/selinux)
- [14.2. タイプ Red Hat Enterprise Linux 7 | Red Hat Customer Portal](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-managing_confined_services-the_apache_http_server-types)

```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/wordpress(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/srv/wordpress/wp-content(/.*)?'
sudo restorecon -RF /srv/wordpress/
ls -Z -a /srv/wordpress/
```

## MariaDB のデータベースダンプの取得とリストア
```bash
mysqldump -u wpoki2a24user -p wpoki2a24db > wpoki2a24db.sql
mysql -u wpoki2a24user -p wpoki2a24db < wpoki2a24db.sql
```

## Let'encrypt を certbot で設定
- [Certbot - Centosrhel7 Nginx](https://certbot.eff.org/lets-encrypt/centosrhel7-nginx)

```bash
sudo certbot certonly --nginx
# nginx 設定ファイルを編集
echo "0 0,12 * * * root python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew" | sudo tee -a /etc/crontab > /dev/null
```

/etc/nginx/conf.d/default.conf への編集内容

```conf
    ssl_certificate /etc/letsencrypt/live/oki2a24.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/oki2a24.com/privkey.pem;
```

## mariadb がフリーズする
 Status: "Waiting for page cleaner"
になる。

```bash
[centos@ip-172-26-1-202 ~]$ sudo systemctl status mysqld
● mariadb.service - MariaDB 10.4.10 database server
   Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/mariadb.service.d
           └─migrated-from-my.cnf-settings.conf
   Active: activating (auto-restart) (Result: signal) since 水 2019-12-11 07:16:48 JST; 222ms ago
     Docs: man:mysqld(8)
           https://mariadb.com/kb/en/library/systemd/
  Process: 25670 ExecStartPost=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
  Process: 18557 ExecStart=/usr/sbin/mysqld $MYSQLD_OPTS $_WSREP_NEW_CLUSTER $_WSREP_START_POSITION (code=killed, signal=ABRT)
  Process: 18509 ExecStartPre=/bin/sh -c [ ! -e /usr/bin/galera_recovery ] && VAR= ||   VAR=`/usr/bin/galera_recovery`; [ $? -eq 0 ]   && systemctl set-environment _WSREP_START_POSITION=$VAR || exit 1 (code=exited, status=0/SUCCESS)
  Process: 18507 ExecStartPre=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
 Main PID: 18557 (code=killed, signal=ABRT)
   Status: "Waiting for page cleaner"

12月 11 07:16:48 ip-172-26-1-202.ap-northeast-1.compute.internal systemd[1]: mariad...
12月 11 07:16:48 ip-172-26-1-202.ap-northeast-1.compute.internal systemd[1]: Failed...
12月 11 07:16:48 ip-172-26-1-202.ap-northeast-1.compute.internal systemd[1]: Unit m...
12月 11 07:16:48 ip-172-26-1-202.ap-northeast-1.compute.internal systemd[1]: mariad...
Hint: Some lines were ellipsized, use -l to show in full.
[centos@ip-172-26-1-202 ~]$
```

[CentOS 7～8 MariaDB インストール、及び初期設定 - eTuts+ Server Tutorial](https://server.etutsplus.com/centos-7-mariadb-install-and-mysql-secure-installation/) に、 MySQL 5.6.8 以前に付属していた設定オプションファイルが掲載されている。
低スペックマシン向けの設定 my-small.cnf を、 InnoDB のコメントを外して使用した。

1日くらいなら、サーバダウンしないようになった。

---

今動いている設定。これを使いたい。

```conf:default.conf
root   /srv/wordpress/;
index  index.php index.html index.htm;

# キャッシュしたファイルが保管されるパスと、キャッシュゾーンの名前、容量を指定
# キャッシュ保管ディレクトリ
# keys_zone=name:size メタデータを格納する共有メモリ領域と、そのサイズ
# levels キャッシュを保存するサブディレクトリ階層の深さ
# inactive アクセスの無いキャッシュを削除するまでの期間
# max_size ゾーン内に保存できるキャッシュ最大値
proxy_cache_path  /var/cache/nginx keys_zone=czone:32m levels=1:2 inactive=3d max_size=256m;

server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;
    server_name oki2a24.com;
    return 301 https://$host$request_uri;
}

# プロキシサーバ設定
server {
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

    # 利用するキャッシュゾーンの名前を指定
    proxy_cache czone;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    # キャッシュされた指定レスポンスコードのコンテンツの有効期限
    proxy_cache_valid 200 404 1d;

    # ドットファイルへのアクセスを禁止、ログへの記録オフ
    location ~ /\. {deny all; access_log off; log_not_found off; }
    # robots.txt へのアクセスはログへの記録オフ
    location = /robots.txt  { access_log off; log_not_found off; }
    # favicon へのアクセスはログへの記録オフ
    location = /favicon.ico { access_log off; log_not_found off; }
    # JavaScript CSS 画像へのアクセスはログへの記録オフ、直ちにプロキシに通しキャッシュ。
    # この期の設定でアクセス元の状態で複数キャッシュを行うが画像ファイルなどは複数キャッシュさせない
    location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
        log_not_found off;
        proxy_pass http://unix:/var/run/nginx.sock;
    }

    # プロキシキャッシュ設定
    # 初期値はキャッシュするにセット
    set $do_not_cache 0;

    # GET 時以外はキャッシュしない
    if ($request_method != "GET") {
        set $do_not_cache 1;
    }

    # .php ファイルへ直接アクセスがあるのは基本的に管理ページのみのためキャッシュしない
    if ($uri ~* "\.php$") {
        set $do_not_cache 1;
    }

    # クッキーから、コメント書き込み中、ログイン中、パスワード保護コンテンツはキャッシュしない
    if ($http_cookie ~ ^.*(comment_author_|wordpress_logged_in|wp-postpass_).*$) {
        set $do_not_cache 1;
    }

    # 今まで組み立てたプロキシ設定でキャッシュを実行
    location / {
        proxy_no_cache $do_not_cache;
        proxy_cache_bypass $do_not_cache;
        proxy_cache_key "$scheme://$host$request_uri";
        proxy_pass http://unix:/var/run/nginx.sock;
    }

}

server {
    # ポートを指定
    listen unix:/var/run/nginx.sock;
    # WordPress カスタム パーマネントリンク対応
    try_files $uri $uri/ /index.php?q=$uri&$args;

    # PHP-FPM 設定
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


    # 実ファイルがない場合のアクセスファイル
    #try_files $uri $uri/ /index.php;

    #location / {
    #    # WordPress パーマリンク設定を利用可能にする
    #    if (!-e $request_filename) {
    #        rewrite ^.+?(/wp-.*) $1 last;
    #        rewrite ^.+?(/.*\.php)$ $1 last;
    #        # ドキュメントルートから WordPress までの相対パス
    #        # (ドキュメントルートにインストールしたため相対パスは記入なし)
    #        rewrite ^ /index.php last;
    #    }
    #}

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    fastcgi_pass   unix:/var/run/php-fpm/php-fpm.sock;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```
