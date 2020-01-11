# ansible_oki2a24.com_wordpress_lemp
oki2a24.com 用の WordPress の LEMP サーバを構築する Ansible プレイブックです。

## 疎通確認
`./hosts` を編集してください。接続先情報を記述します。

```bash
ansible-playbook -i hosts test.yml
ansible-playbook -i hosts site.yml -v --tags common
```

## 実行
```bash
ansible-playbook -i hosts site.yml -v --tags common
```

`./group_vars/all` の change_here 等を適宜編集してください。

## WordPress ディレクトリに SELinux 設定を行う
SELinux は有効にしていますので、 WordPress を `/srv/wordpress/` に配置後、次のコマンドで PHP に実行やファイルの読み書き権限を与える必要があります。

```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/wordpress(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/srv/wordpress/wp-content(/.*)?'
sudo restorecon -RF /srv/wordpress/
```

確認は次のコマンドで可能です。

```bash
ls -Z -a /srv/wordpress/
```
