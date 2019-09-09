- Ansibleが使われるようになった背景
    - 仮想化をベースとしたクラウドの登場により、インフラのライフサイクルが短くなった
        - 構成管理を手動で行うと運用コストが肥大化する

---

- インスタンスのセットアップ

```
// ローカルMacにcontrollerの外部IPを登録(表示はダミー)
kiyota-MacBook-Pro:~ kiyotatakeshi$ cat /private/etc/hosts

35.233.43.126 controller

kiyota-MacBook-Pro:~ kiyotatakeshi$ ssh controller
Last login: Sun Sep  8 09:36:59 2019 from 43x232x216x212.ap43.ftth.ucom.ne.jp

```

- コントローラノードにターゲットの内部IPを登録
    - コントローラインスタンスの秘密鍵をターゲットノードに登録しておく

```

[kiyotatakeshi@ansible-controller ~]$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

# ansible hosts
10.146.0.7 inside1
10.146.0.8 inside2

10.146.0.9 outside1
10.146.0.10 outside2
10.146.0.11 outside3

```

- 接続を確認

```
[kiyotatakeshi@ansible-controller ~]$ ssh inside1
Last login: Sun Sep  8 06:50:09 2019 from 10.146.0.6
[kiyotatakeshi@ansible-inside01 ~]$ exit

[kiyotatakeshi@ansible-controller ~]$ ssh inside2
Last login: Sun Sep  8 06:50:14 2019 from 10.146.0.6
[kiyotatakeshi@ansible-inside02 ~]$ exit

[kiyotatakeshi@ansible-controller ~]$ ssh outside1
Last login: Sun Sep  8 06:50:22 2019 from 10.146.0.6
[kiyotatakeshi@ansible-outside1 ~]$ exit

[kiyotatakeshi@ansible-controller ~]$ ssh outside2
Last login: Sun Sep  8 06:50:26 2019 from 10.146.0.6
[kiyotatakeshi@ansible-outside2 ~]$ exit

[kiyotatakeshi@ansible-controller ~]$ ssh outside3
Last login: Sun Sep  8 06:50:31 2019 from 10.146.0.6
[kiyotatakeshi@ansible-outside3 ~]$ exit

```

---

- controllerにansibleをインストール

```
[kiyotatakeshi@ansible-controller ~]$ sudo yum install ansible

インストール:
  ansible.noarch 0:2.8.4-1.el7

依存性関連をインストールしました:
  PyYAML.x86_64 0:3.10-11.el7
  libyaml.x86_64 0:0.1.4-11.el7_0
  python-babel.noarch 0:0.9.6-8.el7
  python-cffi.x86_64 0:1.6.0-5.el7
  python-enum34.noarch 0:1.0.4-1.el7
  python-httplib2.noarch 0:0.9.2-1.el7
  python-idna.noarch 0:2.4-1.el7
  python-jinja2.noarch 0:2.7.2-3.el7_6
  python-markupsafe.x86_64 0:0.11-10.el7
  python-paramiko.noarch 0:2.1.1-9.el7
  python-ply.noarch 0:3.4-11.el7
  python-pycparser.noarch 0:2.14-1.el7
  python2-cryptography.x86_64 0:1.7.2-2.el7
  python2-jmespath.noarch 0:0.9.0-3.el7
  sshpass.x86_64 0:1.06-2.el7

[kiyotatakeshi@ansible-controller ~]$ ansible --version
ansible 2.8.4
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/kiyotatakeshi/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Jun 20 2019, 20:27:34) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]

```

- VScode上でcotrollerのファイルを編集
    - [Remote SSH を使用](https://blog.masahiko.info/entry/2019/06/15/202003)

```
// Configに以下の内容を記載

Host kiyotatakeshi@controller
  HostName controller
  User kiyotatakeshi
  IdentityFile ~/.ssh/id_rsa

```

- 作業ディレクトリの作成

```
$ tree practical_guide/
practical_guide/
└── test_sec
    └── inventory

```

- configの作成

```
[kiyotatakeshi@ansible-controller ~]$ cat ~/.ansible.cfg
[defaults]
# basic default values

# ターゲットノードの並列処理を行うプロセス数
forks = 15
log_path = $HOME/.ansible/ansible.log

# SSH接続時のフィンガープリントチェックを行わない
host_key_checklog = False

# 新規に接続した場合のみターベットノードの詳細情報取得を行う
gathering = smart

```

---

- ターゲットを設定しコマンドを実行

```
kiyotatakeshi@ansible-controller test_sec]$ pwd
/home/kiyotatakeshi/practical_guide/test_sec

[kiyotatakeshi@ansible-controller test_sec]$ cat inventory/test_inventory.ini
[test_servers]
inside1 ansible_host=10.146.0.7
inside2 ansible_host=10.146.0.8

[control]
local ansible_connection=local


[kiyotatakeshi@ansible-controller test_sec]$ ansible -i inventory/test_inventory.ini test_servers -m ping

inside2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
inside1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
[kiyotatakeshi@ansible-controller test_sec]$ ansible -i inventory/test_inventory.ini control -m ping

local | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```

---

- playbookの実行


```
[kiyotatakeshi@ansible-controller test_sec]$ cat test_playbook.yaml

---
- hosts: test_servers
  become: true
  tasks:
  - name: create directory
    file:
      path: /home/ansible/tmp
      state: directory
      mode: 0755

  - name: copy file
    copy:
      src: ~/test.txt
      dest: /home/ansible/tmp/hosts
      mode: 0644

// 実行
[kiyotatakeshi@ansible-controller test_sec]$ ansible-playbook -i inventory/test_inventory.ini test_playbook.yaml

PLAY [test_servers] ********************************************************************************************

TASK [Gathering Facts] *****************************************************************************************
ok: [inside2]
ok: [inside1]

TASK [create directory] ****************************************************************************************
changed: [inside1]
changed: [inside2]

TASK [copy file] ***********************************************************************************************
changed: [inside1]
changed: [inside2]

PLAY RECAP *****************************************************************************************************
inside1                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
inside2                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

- 確認

```
[kiyotatakeshi@ansible-controller test_sec]$ ssh inside1
Last login: Sun Sep  8 23:06:20 2019 from 10.146.0.6

[kiyotatakeshi@ansible-inside01 ~]$ sudo su -
最終ログイン: 2019/09/08 (日) 23:08:10 UTC日時 pts/0
[root@ansible-inside01 ~]#

[root@ansible-inside01 ~]# ls -ld /home/ansible/tmp
drwxr-xr-x. 2 root root 19 Sep  8 23:04 /home/ansible/tmp

[root@ansible-inside01 ~]# ls -l /home/ansible/tmp/hosts
-rw-r--r--. 1 root root 0 Sep  8 23:04 /home/ansible/tmp/hosts

// 同様に
[kiyotatakeshi@ansible-controller test_sec]$ ssh inside2

```

- オペレーションをシンプルにコード化しメンバーが共通の認識で作業を行えるノウハウこそが、自動化推進の第一歩