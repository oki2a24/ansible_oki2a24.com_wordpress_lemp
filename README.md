# ansible_oki2a24.com_wordpress_lemp
oki2a24.com 用の WordPress の LEMP サーバを構築する Ansible プレイブックです。

## 疎通確認
`./hosts` を編集してください。接続先情報を記述します。次の公式ページを必要に応じて参照してください。

- [How to build your inventory — Ansible Documentation](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html)

```bash
ansible-playbook -i hosts test.yml
```

## 実行
`./group_vars/all` の change_here 等を適宜編集してください。

パッケージアップデート等のタスク実行後、リブートします。

```bash
ansible-playbook -i hosts site.yml --tags first
```

その後、以降のタスクを行ってください。

```bash
ansible-playbook -i hosts site.yml
```

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

## WordPress 本体のアップグレードを行うための一時的な SELinux 設定
下記のコマンドで一時的に WordPress 本体のアップグレードを可能にします。アップグレード完了後は、 WordPress ディレクトリに SELinux 設定を行う、を実行してください。

```bash
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/srv/wordpress(/.*)?'
sudo restorecon -RF /srv/wordpress/
```
