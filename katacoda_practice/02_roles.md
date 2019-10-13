## 環境の準備

```
[root@b7b1b6395e82 ~]# ansible --version
ansible 2.8.5

# cat lab_setup.sh
#!/bin/bash

PASSWORD=password

yum install -y ansible jq tree

echo "[web]" > inventory

for i in 1 2 3
do
    docker run -d --security-opt label:disable --cap-add SYS_ADMIN --name node-${i} -h node-${i} -p808${i}:80 irixjp/katacoda:latest /sbin/init
    sleep 3
    docker exec -it node-${i}  sh -c "systemctl start sshd"
    docker exec -it node-${i}  sh -c "echo ${PASSWORD:?} | passwd --stdin root"
    IPADDR=`docker inspect node-${i}  | jq -r ".[0].NetworkSettings.IPAddress"`
    echo node-${i} ansible_ssh_host=${IPADDR:?} ansible_ssh_user=root ansible_ssh_pass=${PASSWORD:?} >> inventory
done

echo "Exit!!"

# bash ./lab_setup.sh

Changing password for user root.
passwd: all authentication tokens updated successfully.
Exit!!

# cat ./inventory
[web]
node-1 ansible_ssh_host=172.20.0.2 ansible_ssh_user=root ansible_ssh_pass=password
node-2 ansible_ssh_host=172.20.0.3 ansible_ssh_user=root ansible_ssh_pass=password
node-3 ansible_ssh_host=172.20.0.4 ansible_ssh_user=root ansible_ssh_pass=password

```

- playbookの編集

```
# cat site.yml

---
- hosts: web
  name: This is a play within a playbook
  become: yes
  vars:
    httpd_packages:
      - httpd
      - mod_wsgi
    apache_test_message: This is a test message
    apache_max_keep_alive_requests: 115

  tasks:
    - name: install httpd packages
      yum:
        name: "{{ item }}" // リストのアイテムを展開することを宣言
        state: present

      // ループの本体、httpd_packagesに含まれている全てのitemにtasksを実行
      loop: "{{ httpd_packages }}"

      // ハンドラ部分
      notify: restart apache service

    - name: create site-enabled directory
      file:
        name: /etc/httpd/conf/sites-enabled
        state: directory

    - name: copy httpd.conf
      template:
        src: httpd.conf.j2
        dest: /etc/httpd/conf/httpd.conf

      // nameを使用してhandlerを呼び出す
      notify: restart apache service

    - name: copy index.html

      // サーバにファイルを配置
      // その際に、テンプレートエンジンを使用して変数に置換
      template:
        src: index.html.j2
        dest: /var/www/html/index.html

    - name: start httpd
      service:
        name: httpd
        state: started
        enabled: yes


  handlers:
    - name: restart apache service
      service:
        name: httpd
        state: restarted
        enabled: yes

```

- j2テンプレートで置換される内容

```
# grep 'apache_test_message' index.html.j2
    <p>{{ apache_test_message }}</p>

```

- playbookのシンタックスチェック

```
# ansible-playbook --syntax-check site.yml

playbook: site.yml

# ansible-playbook site.yml

PLAY RECAP *******************************************************************************************
node-1                     : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0 ignored=0
node-2                     : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0 ignored=0
node-3                     : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0 ignored=0

```

---
## リファクタリングしてRole化する

- 作業単位で自動化をパーツ化して再利用可能にする

- rolesディレクトリを作成

```
# mkdir -p ~/apache-basic-playbook/roles
# cd ~/apache-basic-playbook/roles
# ansible-galaxy init apache-simple
- apache-simple was created successfully

# tree ~/apache-basic-playbook
/root/apache-basic-playbook
└── roles
    └── apache-simple
        ├── defaults
        │   └── main.yml
        ├── files
        ├── handlers
        │   └── main.yml
        ├── meta
        │   └── main.yml
        ├── README.md
        ├── tasks
        │   └── main.yml
        ├── templates
        ├── tests
        │   ├── inventory
        │   └── test.yml
        └── vars
            └── main.yml

10 directories, 8 files

```

- 使用しないディレクトリを削除

```
# cd ~/apache-basic-playbook/roles/apache-simple/
# pwd
/root/apache-basic-playbook/roles/apache-simple
# rm -rf files tests

# ll
total 28
drwxr-xr-x. 2 root root 4096 Oct 13 07:14 defaults
drwxr-xr-x. 2 root root 4096 Oct 13 07:14 handlers
drwxr-xr-x. 2 root root 4096 Oct 13 07:14 meta
-rw-r--r--. 1 root root 1328 Oct 13 07:14 README.md
drwxr-xr-x. 2 root root 4096 Oct 13 07:14 tasks
drwxr-xr-x. 2 root root 4096 Oct 13 07:14 templates
drwxr-xr-x. 2 root root 4096 Oct 13 07:14 vars
```

- role呼び出し元のplaybookを作成

```
# cd ~/apache-basic-playbook

# ll
total 4
drwxr-xr-x. 3 root root 4096 Oct 13 07:25 roles

# vim site.yml
# cat site.yml
---
- hosts: web
  name: This is my role-based playbook
  become: yes

  roles:
    - apache-simple

```

- roleで使われるデフォルトの変数の設定

```
# vim roles/apache-simple/defaults/main.yml

# cat roles/apache-simple/defaults/main.yml
---
# defaults file for apache-simple
apache_test_message: This is a test message
apache_max_keep_alive_requests: 115

```

- role に特化した変数を定義

```
# vim roles/apache-simple/vars/main.yml

# cat roles/apache-simple/vars/main.yml
---
# vars file for apache-simple
httpd_packages:
  - httpd
  - mod_wsgi
```

- handlerを作成

```
# vim roles/apache-simple/handlers/main.yml

# cat roles/apache-simple/handlers/main.yml
---
# handlers file for apache-simple
- name: restart apache service
  service:
    name: httpd
    state: restarted
    enabled: yes

```

- rolesにtaskを定義

```
# vim roles/apache-simple/tasks/main.yml

# cat roles/apache-simple/tasks/main.yml
---
# tasks file for apache-simple
- name: install httpd packages
  yum:
    name: "{{ item }}"
    state: present
  with_items: "{{ httpd_packages }}"
  notify: restart apache service

- name: create site-enabled directory
  file:
    name: /etc/httpd/conf/sites-enabled
    state: directory

- name: copy httpd.conf
  template:
    src: templates/httpd.conf.j2
    dest: /etc/httpd/conf/httpd.conf
  notify: restart apache service

- name: copy index.html
  template:
    src: templates/index.html.j2
    dest: /var/www/html/index.html

- name: start httpd
  service:
    name: httpd
    state: started
    enabled: yes

```

- テンプレートファイルを配置

```
# pwd
/root/apache-basic-playbook

# ll
total 8
drwxr-xr-x. 3 root root 4096 Oct 13 07:25 roles
-rw-r--r--. 1 root root  100 Oct 13 07:27 site.yml

# cp ~/index.html.j2 roles/apache-simple/templates/
# cp ~/httpd.conf.j2 roles/apache-simple/templates/

# cp ~/inventory ~/apache-basic-playbook
# cp ~/ansible.cfg ~/apache-basic-playbook

# tree ~/apache-basic-playbook
/root/apache-basic-playbook
├── ansible.cfg
├── inventory
├── roles
│   └── apache-simple
│       ├── defaults
│       │   └── main.yml
│       ├── handlers
│       │   └── main.yml
│       ├── meta
│       │   └── main.yml
│       ├── README.md
│       ├── tasks
│       │   └── main.yml
│       ├── templates
│       │   ├── httpd.conf.j2
│       │   └── index.html.j2
│       └── vars
│           └── main.yml
└── site.yml

8 directories, 11 files

```

- configを確認

```
# cat ansible.cfg
[defaults]
inventory         = inventory
host_key_checking = False
force_color       = True
forks             = 1

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null

# cat inventory
[web]
node-1 ansible_ssh_host=172.20.0.2 ansible_ssh_user=root ansible_ssh_pass=password
node-2 ansible_ssh_host=172.20.0.3 ansible_ssh_user=root ansible_ssh_pass=password
node-3 ansible_ssh_host=172.20.0.4 ansible_ssh_user=root ansible_ssh_pass=password

```

-

```
# ansible-playbook --syntax-check site.yml

playbook: site.yml

# ansible-playbook site.yml

PLAY RECAP *****************************************************************************************
node-1                     : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
node-2                     : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
node-3                     : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

- 変数を上書きして再度、playbookを実行
  - ノードのページの内容が更新される

```
# ansible-playbook site.yml -e apache_test_message=vars_from_extra

TASK [apache-simple : copy index.html] *************************************************************
changed: [node-1]
changed: [node-2]
changed: [node-3]

PLAY RECAP *****************************************************************************************
node-1                     : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
node-2                     : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
node-3                     : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```