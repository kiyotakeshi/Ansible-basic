### flaskがレスポンスを返すよう設定

- pythonアプリケーションを表示するためにモジュールを有効化する

```
$ cat webserver.yaml
---
- hosts: webserver
  become: true
  tasks:
    - name: install web components
      apt:
        state: present
        update_cache: yes
        name: "{{ packages }}"
      vars:
        packages:
          - apache2
          - libapache2-mod-wsgi
          - python-pip
          - python-virtualenv

    - name: ensure apache2 started
      service: name=apache2 state=started enabled=yes

    - name: ensure mod_wsgi enabled
      apache2_module: state=present name=wsgi

      # モジュールの読み込みに成功した場合のみ、再起動する
      notify: restart apache2

  handlers:
    - name: restart apache2
      service: name=apache2 state=restarted

$ ansible-playbook webserver.yaml

PLAY [webserver] ***************************************************************************

TASK [Gathering Facts] *********************************************************************
ok: [app01]
ok: [app02]

TASK [install web components] **************************************************************
 [WARNING]: Could not find aptitude. Using apt-get instead

ok: [app01]
ok: [app02]

TASK [ensure apache2 started] **************************************************************
ok: [app01]
ok: [app02]

TASK [ensure mod_wsgi enabled] *************************************************************
ok: [app01]
ok: [app02]

PLAY RECAP *********************************************************************************
app01                      : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
app02                      :

```

---
- python(Flask)のデモアプリのデプロイ

```
$ tree
.
├── ansible.cfg
├── control.yaml
├── database.yaml
├── demo
│   ├── app
│   │   ├── demo.py
│   │   ├── demo.wsgi
│   │   └── requirements.txt
│   └── demo.conf
├── dev
├── loadbalancer.yaml
├── playbooks
│   ├── hostname.yaml
│   └── stack_restart.yaml
└── webserver.yaml

3 directories, 12 files
kiyota-MacBook-Pro:ansible kiyotatakeshi$ tree demo/
demo/
├── app
│   ├── demo.py
│   ├── demo.wsgi
│   └── requirements.txt
└── demo.conf

// webサーバにflaskのコンテンツをデプロイする
$ cat webserver.yaml
---
- hosts: webserver
  become: true
  tasks:
    - name: install web components
      apt:
        state: present
        update_cache: yes
        name: "{{ packages }}"
      vars:
        packages:
          - apache2
          - libapache2-mod-wsgi
          - python-pip
          - python-virtualenv

    - name: ensure apache2 started
      service: name=apache2 state=started enabled=yes

    - name: ensure mod_wsgi enabled
      apache2_module: state=present name=wsgi
      notify: restart apache2

    - name: copy demo app source
      copy: src=demo/app/ dest=/var/www/demo mode=0755
      notify: restart apache2

    - name: copy apache virtual host config
      copy: src=demo/demo.conf dest=/etc/apache2/sites-available mode=0755
      notify: restart apache2

  handlers:
    - name: restart apache2
      service: name=apache2 state=restarted

```

- playbookの実行

```
$ ansible-playbook webserver.yaml

PLAY [webserver] ***************************************************************************

TASK [Gathering Facts] *********************************************************************
ok: [app01]
ok: [app02]

TASK [install web components] **************************************************************
 [WARNING]: Could not find aptitude. Using apt-get instead

ok: [app01]
ok: [app02]

TASK [ensure apache2 started] **************************************************************
ok: [app01]
ok: [app02]

TASK [ensure mod_wsgi enabled] *************************************************************
ok: [app01]
ok: [app02]

// 初めて実行するコマンドのため changed
TASK [copy demo app source] ****************************************************************
changed: [app01]
changed: [app02]

TASK [copy apache virtual host config] *****************************************************
changed: [app01]
changed: [app02]

RUNNING HANDLER [restart apache2] **********************************************************
changed: [app01]
changed: [app02]

PLAY RECAP *********************************************************************************
app01                      : ok=7    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
app02                      : ok=7    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

---
- pip環境をセット

```
// webserver.yamlのtasksに追加

    - name: setup python virtualenv
      pip: requirements=/var/www/demo/requirements.txt virtualenv=/var/www/demo/.venv
      notify: restart apache2

$ ansible-playbook webserver.yaml

(略)
TASK [setup python virtualenv] *************************************************************
changed: [app01]
changed: [app02]

RUNNING HANDLER [restart apache2] **********************************************************
changed: [app01]
changed: [app02]

PLAY RECAP *********************************************************************************
app01                      : ok=8    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
app02                      : ok=8    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```
---
- apache2のセットアップ

```
// 確認

$ ssh 33.113.42.146 -p 1994

kiyotatakeshi@ansible-template-3:~$ ls -l /etc/apache2/sites-available/
total 16
-rw-r--r-- 1 root root 1332 Nov  3  2018 000-default.conf
-rw-r--r-- 1 root root 6338 Jun 16 09:49 default-ssl.conf
-rwxr-xr-x 1 root root  279 Sep  1 15:57 demo.conf

// デフォルトのapacheのページがシンボリックリンクで設定されている
kiyotatakeshi@ansible-template-3:~$ ls -l /etc/apache2/sites-enabled/
total 0
lrwxrwxrwx 1 root root 35 Sep  1 12:17 000-default.conf -> ../sites-available/000-default.conf

// webserver.yaml のtaskを追加

    - name: de-activate default apache site
      file: path=/etc/apache2/sites-enabled/000-default.conf state=absent
      notify: restart apache2

    - name: activate demo apache site

      # シンボリックリンクをはる
      file: src=/etc/apache2/sites-available/demo.conf dest=/etc/apache2/sites-enabled/demo.conf state=link
      notify: restart apache2

// 実行
$ ansible-playbook webserver.yaml

(略)

TASK [de-activate default apache site] **************************************************************************************
ok: [app01]
ok: [app02]

TASK [activate demo apache site] ********************************************************************************************
changed: [app01]
changed: [app02]

RUNNING HANDLER [restart apache2] *******************************************************************************************
changed: [app01]
changed: [app02]

PLAY RECAP ******************************************************************************************************************
app01                      : ok=10   changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
app02                      : ok=10   changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

- flaskがリクエストを返すことを確認

```
// IPはダミー
$ curl 37.244.72.246 | head -n 15
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    38  100    38    0     0    123      0 --:--:-- --:--:-- --:--:--   124
Hello, from sunny ansible-template-3!

$ curl 35.202.154.247 | head -n 15
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    24  100    24    0     0     39      0 --:--:-- --:--:-- --:--:--    39
Hello, from sunny test!

```

---
- 名前解決を設定を変更(Mac)

```
// 編集後の差分
$ diff -u /private/etc/hosts /private/etc/hosts.org
--- /private/etc/hosts	2019-09-02 02:04:08.000000000 +0900
+++ /private/etc/hosts.org	2019-09-02 02:03:06.000000000 +0900
@@ -7,9 +7,3 @@
 127.0.0.1	localhost
 255.255.255.255	broadcasthost
 ::1             localhost
-
-# ansible practice
-32.239.55.249 lb01
-38.239.62.246 app01
-33.202.214.247 app02
-33.80.14.135 db01

// キャッシュをリセット
$ dscacheutil -flushcache

$ curl app01
Hello, from sunny ansible-template-3!

$ curl app02
Hello, from sunny test!

$ curl lb01 | head -n 10
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0  39979      0 --:--:-- --:--:-- --:--:-- 40800
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }

```