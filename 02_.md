### playbookを使った環境の構築

- playbookの実行

```
$ cat loadbalancer.yaml
---
- hosts: loadbalancer
  become: true
  tasks:
    - name: install nginx

      # presentは初回のインストールで最新のものをインストールするが、アップデートはかけない
      # latestにするとアップグレードを自動で行う
      # absentにすると削除
      apt: name=nginx state=present update_cache=yes

$ ls
ansible.cfg		dev			playbooks
database.yaml		loadbalancer.yaml

$ ansible-playbook loadbalancer.yaml
 [WARNING]: Found both group and host with same name: control


PLAY [loadbalancer] **********************************************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [lb01]

TASK [install nginx] *********************************************************************************
 [WARNING]: Updating cache and auto-installing missing dependency: python-apt

 [WARNING]: Could not find aptitude. Using apt-get instead

changed: [lb01]

PLAY RECAP *******************************************************************************************
lb01                       : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

- 対象サーバにて確認

```
// インストールされている
$ sudo dpkg -l nginx
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                Version        Architecture   Description
+++-===================-==============-==============-============================================
ii  nginx               1.10.3-1+deb9u all            small, powerful, scalable web/proxy server

// 削除
$ sudo apt remove --purge nginx
Reading package lists... Done
Building dependency tree
Reading state information... Done

// 削除できている
kiyotatakeshi@ansible-template-1:~$ sudo dpkg -l nginx
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                Version        Architecture   Description
+++-===================-==============-==============-============================================
un  nginx               <none>         <none>         (no description available)

```

- 再度playbookを実行

```
$ ansible-playbook loadbalancer.yaml

PLAY [loadbalancer] **********************************************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [lb01]

TASK [install nginx] *********************************************************************************
 [WARNING]: Could not find aptitude. Using apt-get instead

changed: [lb01]

PLAY RECAP *******************************************************************************************
lb01                       : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

// 接続、IPはダミー
$ ssh 38.213.8.209 -p 1994
Linux ansible-template-1 4.9.0-9-amd64 #1 SMP Debian 4.9.168-1+deb9u8 (2019-08-11) x86_64

// 再度、インストールできている
kiyotatakeshi@ansible-template-1:~$ sudo dpkg -l nginx
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                Version        Architecture   Description
+++-===================-==============-==============-============================================
ii  nginx               1.10.3-1+deb9u all            small, powerful, scalable web/proxy server

```

---
- mysqlのセットアップ

```
$ cat database.yaml
---
- hosts: database
  become: true
  tasks:
    - name: install mysql-server
      apt: name=mysql-server state=present update_cache=yes

$ ansible-playbook database.yaml
 [WARNING]: Found both group and host with same name: control


PLAY [database] **************************************************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [db01]

TASK [install mysql-server] **************************************************************************
 [WARNING]: Updating cache and auto-installing missing dependency: python-apt

 [WARNING]: Could not find aptitude. Using apt-get instead

changed: [db01]

PLAY RECAP *******************************************************************************************
db01                       : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

// 対象に接続し、確認
kiyotatakeshi@ansible-template-2:~$ dpkg -l mysql-server
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                Version        Architecture   Description
+++-===================-==============-==============-============================================
ii  mysql-server        5.5.9999+defau amd64          MySQL database server binaries and system da

```

---
- webserverのセットアップ

```
// 2.8では推奨されない書き方
$ cat webserver.yaml

---
- hosts: webserver
  become: true
  tasks:
  - name: install web components
    apt: name={{item}} state=present update_cache=yes
    with_items:
    - apache2
    - libapache2-mod-wsgi
    - python-pip
    - python-virtualenv

$ ansible-playbook webserver.yaml
 [WARNING]: Found both group and host with same name: control


PLAY [webserver] *************************************************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [app01]
ok: [app02]

// エラーが出ている
// ループの書き方に item は推奨されていない
TASK [install web components] ************************************************************************
[DEPRECATION WARNING]: Invoking "apt" only once while using a loop via squash_actions is deprecated.
Instead of using a loop to supply multiple items and specifying `name: "{{item}}"`, please use `name:
 ['apache2', 'libapache2-mod-wsgi', 'python-pip', 'python-virtualenv']` and remove the loop. This
feature will be removed in version 2.11. Deprecation warnings can be disabled by setting
deprecation_warnings=False in ansible.cfg.
[DEPRECATION WARNING]: Invoking "apt" only once while using a loop via squash_actions is deprecated.
Instead of using a loop to supply multiple items and specifying `name: "{{item}}"`, please use `name:
 ['apache2', 'libapache2-mod-wsgi', 'python-pip', 'python-virtualenv']` and remove the loop. This
feature will be removed in version 2.11. Deprecation warnings can be disabled by setting
deprecation_warnings=False in ansible.cfg.

// インストール自体はできている
changed: [app01] => (item=['apache2', 'libapache2-mod-wsgi', 'python-pip', 'python-virtualenv'])
 [WARNING]: Updating cache and auto-installing missing dependency: python-apt

 [WARNING]: Could not find aptitude. Using apt-get instead

changed: [app02] => (item=['apache2', 'libapache2-mod-wsgi', 'python-pip', 'python-virtualenv'])

PLAY RECAP *******************************************************************************************
app01                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
app02                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

// state=absentにしてインストールしたものを削除
```

- playbookを書き直す

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

      # aptモジュールのドキュメントを参照し、ループの書き方を確認
      name: "{{ packages }}"
    vars:
      packages:
      - apache2
      - libapache2-mod-wsgi
      - python-pip
      - python-virtualenv

// 実行するとエラーがなくなった
$ ansible-playbook webserver.yaml

PLAY [webserver] *************************************************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [app01]
ok: [app02]

TASK [install web components] ************************************************************************
 [WARNING]: Could not find aptitude. Using apt-get instead

changed: [app01]
changed: [app02]

PLAY RECAP *******************************************************************************************
app01                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
app02                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```
---
- 起動設定を追加

```
$ cat loadbalancer.yaml
---
- hosts: loadbalancer
  become: true
  tasks:
    - name: install nginx

      # presentは初回のインストールで最新のものをインストールするが、アップデートはかけない
      # latestにするとアップグレードを自動で行う
      # absentにすると削除
      apt: name=nginx state=present update_cache=yes

    - name: ensure nginx started
      service: name=nginx state=started enabled=yes

$ ansible-playbook loadbalancer.yaml

PLAY [loadbalancer] **********************************************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [lb01]

TASK [install nginx] *********************************************************************************
 [WARNING]: Could not find aptitude. Using apt-get instead

ok: [lb01]

TASK [ensure nginx started] **************************************************************************
ok: [lb01]

PLAY RECAP *******************************************************************************************
lb01                       : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

- nginxが起動していることを確認

```
$ curl 35.243.35.249
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
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

- apacheを起動

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

    # 起動設定を追加
    - name: ensure apache2 started
      service: name=apache2 state=started enabled=yes

$ ansible-playbook webserver.yaml

(略)

TASK [ensure apache2 started] **************************************************************
ok: [app01]
ok: [app02]

PLAY RECAP *********************************************************************************
app01                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
app02                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

- 起動していることを確認

```
$ curl 35.345.81.226 | head -n 15
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 10701  100 10701    0     0   617k      0 --:--:-- --:--:-- --:--:--  653k

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <title>Apache2 Debian Default Page: It works</title>
    <style type="text/css" media="screen">
  * {
    margin: 0px 0px 0px 0px;
    padding: 0px 0px 0px 0px;
  }

  body, html {
    padding: 3px 3px 3px 3px;

```

---
- mysqlを起動

```
$ cat database.yaml
---
- hosts: database
  become: true
  tasks:
    - name: install mysql-server
      apt: name=mysql-server state=present update_cache=yes

    - name: ensure mysql started
      service: name=mysql state=started enabled=yes

$ ansible-playbook database.yaml

(略)

TASK [ensure mysql started] ****************************************************************
ok: [db01]

```