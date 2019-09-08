### flaskからmysqlに接続

- app0[12],lbには外部IPでアクセスするが、dbへはwebサーバから内部IPでリクエストする
    - GCPの場合、**外部IP**,**内部IP**を確認して設定

---

- webserverの設定を変更
    - flaskからmysqlにアクセスするのに必要なツールのインストール

```
      vars:
        packages:
          - apache2
          - libapache2-mod-wsgi
          - python-pip
          - python-virtualenv
          # これを追加
          - python-mysqldb

$ ansible-playbook webserver.yaml

PLAY [webserver] ***************************************************************

TASK [Gathering Facts] *********************************************************
ok: [app01]
ok: [app02]

TASK [install web components] **************************************************
 [WARNING]: Could not find aptitude. Using apt-get instead

changed: [app01]
changed: [app02]

TASK [ensure apache2 started] **************************************************
ok: [app01]
ok: [app02]

TASK [ensure mod_wsgi enabled] *************************************************
ok: [app01]
ok: [app02]

TASK [copy demo app source] ****************************************************
ok: [app01]
ok: [app02]

TASK [copy apache virtual host config] *****************************************
ok: [app01]
ok: [app02]

TASK [setup python virtualenv] *************************************************
ok: [app01]
ok: [app02]

TASK [de-activate default apache site] *****************************************
ok: [app01]
ok: [app02]

TASK [activate demo apache site] ***********************************************
ok: [app01]
ok: [app02]

PLAY RECAP *********************************************************************
app01                      : ok=9    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
app02                      : ok=9    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```
- 確認

```
// インストールできている
$ sudo dpkg -l python-mysqldb
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                    Version          Architecture     Description
+++-=======================-================-================-===================================================
ii  python-mysqldb          1.3.7-1.1        amd64            Python interface to MySQL

```
---

- mysqlのテーブルの設定

```
$ cat database.yaml
---
- hosts: database
  become: true
  tasks:
    - name: install tools
      apt: name={{item}} state=present update_cache=yes
      with_items:
        - python-mysqldb

    - name: install mysql-server
      apt: name=mysql-server state=present update_cache=yes

    - name: ensure mysql started
      service: name=mysql state=started enabled=yes

    - name: ensure mysql listening on all ports
      lineinfile: dest=/etc/mysql/mariadb.conf.d/50-server.cnf regexp=^bind-address
                  line="bind-address = 0.0.0.0"
      notify: restart mysql

    # テーブルの設定を追加
    - name: create demo database
      mysql_db: name=demo state=present

    - name: create demo user
      mysql_user: name=demo password=demo priv=demo.*:ALL host='%' state=present

  handlers:
    - name: restart mysql
      service: name=mysql state=restarted

// playbookの実行
ansible-playbook database.yaml

PLAY [database] *****************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************
ok: [db01]

// 警告が出た
TASK [install tools] ************************************************************************************************
[DEPRECATION WARNING]: Invoking "apt" only once while using a loop via squash_actions is deprecated. Instead of
using a loop to supply multiple items and specifying `name: "{{item}}"`, please use `name: ['python-mysqldb']` and
remove the loop. This feature will be removed in version 2.11. Deprecation warnings can be disabled by setting
deprecation_warnings=False in ansible.cfg.
ok: [db01] => (item=['python-mysqldb'])
 [WARNING]: Could not find aptitude. Using apt-get instead


TASK [install mysql-server] *****************************************************************************************
ok: [db01]

TASK [ensure mysql started] *****************************************************************************************
ok: [db01]

TASK [ensure mysql listening on all ports] **************************************************************************
ok: [db01]

TASK [create demo database] *****************************************************************************************
changed: [db01]

TASK [create demo user] *********************************************************************************************
changed: [db01]

PLAY RECAP **********************************************************************************************************
db01                       : ok=7    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

- 注意のあった箇所を修正

```
  tasks:
    - name: install tools
      apt: name='python-mysqldb' state=present update_cache=yes
      with_items:
        - python-mysqldb

TASK [install tools] ************************************************************************************************
ok: [db01] => (item=python-mysqldb)

```

---
- 接続チェック
    - 以下のやり方では繋がらない
    - トラブルシューティングのため一応残しておく

```
$ curl app01/db
(_mysql_exceptions.OperationalError) (2005, "Unknown MySQL server host 'db01' (-2)")
kiyota-MacBook-Pro:ansible kiyotatakeshi$ curl app02/db
(_mysql_exceptions.OperationalError) (2005, "Unknown MySQL server host 'db01' (-2)")
kiyota-MacBook-Pro:ansible kiyotatakeshi$ curl lb01/db
(_mysql_exceptions.OperationalError) (2005, "Unknown MySQL server host 'db01' (-2)")

// db01の名前解決ができていないため接続できない
```

- app01,app02に名前解決できるように設定を追加

```

$ cat /etc/hosts

127.0.0.1	localhost
::1		localhost ip6-localhost ip6-loopback
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters

# 例によってダミーです
# ansible practice
34.85.74.135 db01
35.203.61.246 app01
35.202.224.247 app02
35.203.25.249 lb01
```

- 詰んだ
    - グローバルIPで設定していたので接続できない

```
$ curl app01/db
(_mysql_exceptions.OperationalError) (2003, 'Can\'t connect to MySQL server on \'db01\' (110 "Connection timed out")')

```

- いろいろとトラブルシューティングしたがわからん

```
// dbはできている

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| demo               |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.02 sec)

MariaDB [(none)]> use demo;
Database changed
MariaDB [demo]> show tables;
Empty set (0.00 sec)


// wsgiも疑った
kiyotatakeshi@ansible-template-3:~$ sudo su -
root@ansible-template-3:~# cat /var/www/demo/demo.wsgi
activate_this = '/var/www/demo/.venv/bin/activate_this.py'
execfile(activate_this, dict(__file__=activate_this))

import os
// db01にIPをハードコーディングしてもダメでした
os.environ['DATABASE_URI'] = 'mysql://demo:demo@db01/demo'

import sys
sys.path.insert(0, '/var/www/demo')

from demo import app as application

// GCP上のファーアウォールルールも確認したが、いまいちわからない
```

---
- 解決(flaskからdbへ疎通)
    - 内部IPで疎通するよう設定を変更

```
$ ssh -p 1994 lb01
$ ssh -p 1994 db01
$ ssh -p 1994 app01
$ ssh -p 1994 app02

$ sudo su -

# vi /etc/hosts

# cat /etc/hosts
127.0.0.1	localhost
::1		localhost ip6-localhost ip6-loopback
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters

# ansible practice
# GCPの内部IPを名前解決できるよう設定
10.146.0.3 lb01
10.146.0.5 app01
10.138.0.2 app02
10.146.0.4 db01

```

- flaskからdbへ接続できることを確認

```
// app0[12],lbは外部からアクセス可能なため、ローカルMacからcurl
$ curl app01/db
Database Connected from ansible-template-3!
$ curl app02/db
Database Connected from test!
$ curl lb01/db
Database Connected from ansible-template-3!
$ curl lb01/db
Database Connected from test!

// ローカルMacは外部IPを名前解決できるようにしておく(再掲)
kiyota-MacBook-Pro:ansible kiyotatakeshi$ cat /private/etc/hosts
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost
255.255.255.255	broadcasthost
::1             localhost

# 外部IPを登録
# ansible practice
35.203.50.229 lb01
35.233.64.216 app01
35.212.204.147 app02
34.81.14.105 db01
```