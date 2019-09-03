### nginxをロードバランサーとして使用する

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

    - name: configure nginx site
      template: src=templates/nginx.conf.j2 dest=/etc/nginx/sites-available/demo mode=0644
      notify: restart nginx

    - name: de-activate default nginx site
      file: path=/etc/nginx/sites-enabled/default state=absent
      notify: restart nginx

    - name: activate default nginx site
      file: src=/etc/nginx/sites-available/demo dest=/etc/nginx/sites-enabled/demo state=link
      notify: restart nginx

  handlers:
    - name: restart nginx
      service: name=nginx state=restarted

// テンプレートを用意(IPはダミー)
$ cat templates/nginx.conf.j2
upstream demo {
{% for server in groups.webserver%}
    server {{ server }};
{% endfor %}
}

server {
    listen 80;

    location / {
        proxy_pass http://demo;
    }
}

// テンプレートはinventoryの中身を読み込んでいる
$ cat dev

[loadbalancer]
lb01 ansible_host=31.213.155.249 ansible_port=1994

[webserver]
app01 ansible_host=31.213.162.246 ansible_port=1994
app02 ansible_host=31.212.254.247 ansible_port=1994

[database]
db01 ansible_host=31.85.74.131 ansible_port=1994

```

- playbookの実行もエラー

```
$ ansible-playbook loadbalancer.yaml
fatal: [lb01]: FAILED! => {"changed": false, "msg": "Unable to start service nginx: Job for nginx.service failed because the control process exited with error code.\nSee \"systemctl status nginx.service\" and \"journalctl -xe\" for details.\n"}

// 対象サーバに入り確認
Sep 02 22:52:54 ansible-template-1 nginx[8688]: nginx: [emerg] host not found in upstream "app01" i
n /etc/nginx/sites-enabled/demo:2
Sep 02 22:52:54 ansible-template-1 nginx[8688]: nginx: configuration file /etc/nginx/nginx.conf tes
t failed
Sep 02 22:52:54 ansible-template-1 systemd[1]: nginx.service: Control process exited, code=
exited status=1
Sep 02 22:52:54 ansible-template-1 systemd[1]: Failed to start A high performance web serve
r and a reverse proxy server.
-- Subject: Unit nginx.service has failed
-- Defined-By: systemd
-- Support: https://www.debian.org/support
--
-- Unit nginx.service has failed.
--

// エラーの原因となっている部分を確認
// app0[12]が設定として無効(名前解決できないから)
kiyotatakeshi@ansible-template-1:~$ cat /etc/nginx/sites-enabled/demo
upstream demo {
    server app01;
    server app02;
}

server {
    listen 80;

    location / {
        proxy_pass http://demo;
    }
}

```

- 名前解決の設定
    - 今回はひとまず手動だが、自動化の方法も考える

```
// loadbalancerのtargetのサーバの名前解決を設定
kiyotatakeshi@ansible-template-1:~$ sudo vi /etc/hosts

kiyotatakeshi@ansible-template-1:~$ cat /etc/hosts
127.0.0.1	localhost
::1		localhost ip6-localhost ip6-loopback
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters

10.146.0.3 ansible-template-1.asia-northeast1-b.c.spheric-crow-251511.internal ansible-template-1  # Added by Google
169.254.169.254 metadata.google.internal  # Added by Google

# loadbalancer target
32.21.62.36 app01
32.21.254.247 app02

```

- 再度playbookを実行
    - 成功(/etc/hostsを編集する仕組みを考える)

```
$ ansible-playbook loadbalancer.yaml

PLAY [loadbalancer] *************************************************************************

TASK [Gathering Facts] **********************************************************************
ok: [lb01]

TASK [install nginx] ************************************************************************
 [WARNING]: Could not find aptitude. Using apt-get instead

ok: [lb01]

TASK [ensure nginx started] *****************************************************************
ok: [lb01]

TASK [configure nginx site] *****************************************************************
changed: [lb01]

TASK [de-activate default nginx site] *******************************************************
ok: [lb01]

TASK [activate default nginx site] **********************************************************
changed: [lb01]

RUNNING HANDLER [restart nginx] *************************************************************
changed: [lb01]

PLAY RECAP **********************************************************************************
lb01                       : ok=7    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

- 負荷分散できていることが確認できる

```
// targetへのcurl
$ curl app01
Hello, from sunny ansible-template-3!

$ curl app02
Hello, from sunny test!

// LBへのcurl
$ curl lb01
Hello, from sunny ansible-template-3!

$ curl lb01
Hello, from sunny test!

```

---
- mysqlの設定
    - 外部から3306へのアクセスを受け付ける

```
// 現状の確認
// localアドレスが設定されている
$ ssh -p 1994 db01

kiyotatakeshi@ansible-template-2:~$ sudo netstat -an
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:1994            0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN

// 設定ファイルを探す
$ sudo grep -r 'bind' /etc/mysql/
/etc/mysql/mariadb.conf.d/50-server.cnf:bind-address		= 127.0.0.1

// バックアップをとっておく
kiyotatakeshi@ansible-template-2:~$ sudo cp /etc/mysql/mariadb.conf.d/50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf.org

```

- playbookの実行

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

    - name: ensure mysql listening on all ports
      lineinfile: dest=/etc/mysql/mariadb.conf.d/50-server.cnf regexp=^bind-address
                  line="bind-address = 0.0.0.0"
      notify: restart mysql

  handlers:
    - name: restart mysql
      service: name=mysql state=restarted

// 実行
$ ansible-playbook database.yaml

PLAY [database] ****************************************************************

TASK [Gathering Facts] *********************************************************
ok: [db01]

TASK [install mysql-server] ****************************************************
 [WARNING]: Could not find aptitude. Using apt-get instead

ok: [db01]

TASK [ensure mysql started] ****************************************************
ok: [db01]

TASK [ensure mysql listening on all ports] *************************************
changed: [db01]

RUNNING HANDLER [restart mysql] ************************************************
changed: [db01]

PLAY RECAP *********************************************************************
db01                       : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

- ポートの確認

```
$  ansible -a "netstat -an" db01

db01 | CHANGED | rc=0 >>
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN

$ curl app01/db
No module named MySQLdb

```