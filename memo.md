### Ansible

---
- インストール

```
$ pip install ansible

```

```
$ ansible --version
ansible 2.8.4
  config file = None
  configured module search path = ['/Users/kiyotatakeshi/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /Users/kiyotatakeshi/.pyenv/versions/3.7.3/lib/python3.7/site-packages/ansible
  executable location = /Users/kiyotatakeshi/.pyenv/versions/3.7.3/bin/ansible
  python version = 3.7.3 (default, Jun 22 2019, 17:21:01) [Clang 10.0.1 (clang-1001.0.46.4)]

$ ansible-playbook --version
ansible-playbook 2.8.4
  config file = None
  configured module search path = ['/Users/kiyotatakeshi/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /Users/kiyotatakeshi/.pyenv/versions/3.7.3/lib/python3.7/site-packages/ansible
  executable location = /Users/kiyotatakeshi/.pyenv/versions/3.7.3/bin/ansible-playbook
  python version = 3.7.3 (default, Jun 22 2019, 17:21:01) [Clang 10.0.1 (clang-1001.0.46.4)]

$ ansible-galaxy --help
Usage: ansible-galaxy [delete|import|info|init|install|list|login|remove|search|setup] [--help] [options] ...

Perform various Role related operations.

Options:
  -h, --help            show this help message and exit
  -c, --ignore-certs    Ignore SSL certificate validation errors.
  -s API_SERVER, --server=API_SERVER
                        The API server destination
  -v, --verbose         verbose mode (-vvv for more, -vvvv to enable
                        connection debugging)
  --version             show program's version number, config file location,
                        configured module search path, module location,
                        executable location and exit

 See 'ansible-galaxy <command> --help' for more information on a specific
command.
```

---
- Inventory

```
// デフォルトで読み込まれるもの

$ cat /etc/ansible/hosts
green.example.com

$ ansible --list-hosts all
  hosts (1):
    green.example.com

```

- プロジェクトごとに設定する

```
$ ls
dev

$ cat dev

[loadbalancer]
lb01

[webserver]
app01
app02

[database]
db01

[control]
control

// inventoryを指定して対象ホストを確認

$ ansible -i dev --list-host all
 [WARNING]: Found both group and host with same name: control

  hosts (5):
    lb01
    app01
    app02
    db01
    control

```

- プロジェクト内で適用される設定ファイルを作成

```
$ ls -l
total 16
-rw-r--r--  1 kiyotatakeshi  staff   28  9  1 01:03 ansible.cfg
-rw-r--r--  1 kiyotatakeshi  staff  105  9  1 01:01 dev

$ cat ansible.cfg
[defaults]
inventory = ./dev

$ ansible --list-hosts all
 [WARNING]: Found both group and host with same name: control

  hosts (5):
    lb01
    app01
    app02
    db01
    control

```

---
- host selection(Patterns)

```
$ ansible --list-hosts "*"
 [WARNING]: Found both group and host with same name: control

  hosts (5):
    lb01
    app01
    app02
    db01
    control

$ ansible --list-hosts loadbalancer
 [WARNING]: Found both group and host with same name: control

  hosts (1):
    lb01

$ ansible --list-hosts webserver
 [WARNING]: Found both group and host with same name: control

  hosts (2):
    app01
    app02

$ ansible --list-hosts "app0*"
 [WARNING]: Found both group and host with same name: control

  hosts (2):
    app01
    app02


// 複数のグループを対象
 $ ansible --list-hosts database,control
 [WARNING]: Found both group and host with same name: control

  hosts (2):
    db01
    control


$ ansible --list-hosts webserver
 [WARNING]: Found both group and host with same name: control

  hosts (2):
    app01
    app02

// グループ内の特定のホストを対象
$ ansible --list-hosts webserver[0]
 [WARNING]: Found both group and host with same name: control

  hosts (1):
    app01

```

---
- コマンドの実行
  - GCP上のインスタンスに対して実行
  - GCP上のインスタンスのセットアップ方法は割愛

```
$ cat /etc/ansible/hosts
35.212.254.200 ansible_port=1994

```

```
$ ansible -m ping all

35.212.254.200 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}

$ ansible -m command -a "hostname" all

35.232.554.247 | CHANGED | rc=0 >>
test

```

---
- playbookの活用

```
// GCPのインスタンスを対象に追加
$ cat dev

[test]
test01 ansible_host=13.112.254.247 ansible_port=1994

[loadbalancer]
lb01

[webserver]
app01
app02

[database]
db01

[control]
control ansible_connection=local

```

- playbookディレクトリ内に実行内容を記載したファイルを作成

```
$ tree
.
├── ansible.cfg
├── dev
└── playbooks
    └── hostname.yaml

1 directory, 3 files

$ cat playbooks/hostname.yaml
---
  - hosts: all
    tasks:
      - command: hostname

```

- 実行

```
$ ansible-playbook playbooks/hostname.yaml
 [WARNING]: Found both group and host with same name: control


PLAY [all] **************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************
fatal: [lb01]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: Could not resolve hostname lb01: nodename nor servname provided, or not known", "unreachable": true}
fatal: [app01]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: Could not resolve hostname app01: nodename nor servname provided, or not known", "unreachable": true}
fatal: [app02]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: Could not resolve hostname app02: nodename nor servname provided, or not known", "unreachable": true}
fatal: [db01]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: Could not resolve hostname db01: nodename nor servname provided, or not known", "unreachable": true}
ok: [control]
ok: [test01]

TASK [command] **********************************************************************************************************
changed: [control]
changed: [test01]

PLAY RECAP **************************************************************************************************************
app01                      : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
app02                      : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
control                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
db01                       : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
lb01                       : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
test01                     : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

- taskの表示を変えて、何を実行しているかわかりやすくする

```
$ cat playbooks/hostname.yaml
---
  - hosts: all
    tasks:
      - name: get server hostname
        command: hostname

$ ansible-playbook playbooks/hostname.yaml
 [WARNING]: Found both group and host with same name: control

// nameで指定した表示になる
TASK [get server hostname] **********************************************************************************************
changed: [control]
changed: [test01]

```

---
- GCP上に4台のホストを用意し、playbookを実行

```
// ディレクトリ構成の確認
$ tree
.
├── ansible.cfg
├── dev
└── playbooks
    └── hostname.yaml

1 directory, 3 files

// GCP上のホストの情報を対象として登録
// IPはダミーの物に変えてあり、sshアクセスポートは1994番に変更
$ cat dev

[loadbalancer]
lb01 ansible_host=24.85.74.125 ansible_port=1994

[webserver]
app01 ansible_host=25.212.264.247 ansible_port=1994
app02 ansible_host=26.242.126.148 ansible_port=1994

[database]
db01 ansible_host=26.242.117.61 ansible_port=1994

[control]
control ansible_connection=local

// playbookの確認
$ cat playbooks/hostname.yaml
---
  - hosts: all
    tasks:
      - name: get server hostname
        command: hostname

```

- 実行

```
$ ansible-playbook playbooks/hostname.yaml
 [WARNING]: Found both group and host with same name: control


PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [lb01]
ok: [db01]
ok: [app02]
ok: [control]
ok: [app01]

TASK [get server hostname] *****************************************************
changed: [control]
changed: [db01]
changed: [lb01]
changed: [app02]
changed: [app01]

PLAY RECAP *********************************************************************
app01                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
app02                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
control                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
db01                       : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
lb01                       : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```
