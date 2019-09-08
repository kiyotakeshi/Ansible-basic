### コンテンツへのアクセスチェックも自動化する

- curlがうまくいかず断念

---
- 再起動の設定を入れる

```
$ cat playbooks/stack_restart.yaml
---
# Bring stack down
- hosts: loadbalancer
  become: true
  tasks:
    - service: name=nginx state=stopped

- hosts: webserver
  become: true
  tasks:
    - service: name=apache2 state=stopped
    - wait_for: port=80 state=drained

# Restart mysql
- hosts: database
  become: true
  tasks:
    - service: name=mysql state=stopped
    - wait_for: port=3306 state=started

# Bring stack up
- hosts: webserver
  become: true
  tasks:
    - service: name=apache2 state=started
    - wait_for: port=80

- hosts: loadbalancer
  become: true
  tasks:
    - service: name=nginx state=started
    - wait_for: port=80

```

- lbからコンテンツを確認する際に必要？

```
kiyota-MacBook-Pro:ansible kiyotatakeshi$ cat loadbalancer.yaml

  tasks:
    - name: install tools
      apt: name='python-httplib2' state=present update_cache=yes

$ ansible-playbook loadbalancer.yaml

PLAY [loadbalancer] *************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************
ok: [lb01]

TASK [install tools] ************************************************************************************************
 [WARNING]: Could not find aptitude. Using apt-get instead

changed: [lb01]

TASK [install nginx] ************************************************************************************************
ok: [lb01]

TASK [ensure nginx started] *****************************************************************************************
ok: [lb01]

TASK [configure nginx site] *****************************************************************************************
ok: [lb01]

TASK [de-activate default nginx site] *******************************************************************************
ok: [lb01]

TASK [activate default nginx site] **********************************************************************************
ok: [lb01]

PLAY RECAP **********************************************************************************************************
lb01                       : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

- 再起動後に実行するコンテンツの確認playbookを作成

```
$ cat playbooks/stack_status.yaml
---
- hosts: loadbalancer
  become: true
  tasks:
    - name: verify nginx service
      command: service nginx status

    - name: verify nginx is listening on 80
      wait_for: port=80 timeout=1

- hosts: webserver
  become: true
  tasks:
    - name: verify apache2 service
      command: service apache2 status

    - name: verify apache2 is listening on 80
      wait_for: port=80 timeout=1

- hosts: database
  become: true
  tasks:
    - name: verify mysql service
      command: service mysql status

    - name: verify mysql is listening on 3306
      wait_for: port=3306 timeout=1

// ここがうまくいかない
- hosts: loadbalancer
  tasks:
    - name: verify backend index response
      uri: url=http://{{item}} return_content=yes
      with_items: "{{groups.webserver}}"
      register: app_index

    - fail: msg="index failed to return content"
      when: "'Hello, from sunny {{item.item}}!' not in item.content"
      with_items: "{{app_index.results}}"

    - name: verify backend db response
      uri: url=http://{{item}}/db return_content=yes
      with_items: "{{webserver.*}}"
      register: app_db

    - fail: msg="db failed to return content"
      when: "'Database Connected from {{item.item}}!' not in item.content"
      with_items: "{{app_db.results}}"

```

- うまくいかない

```
$ ansible-playbook playbooks/stack_status.yaml

PLAY [loadbalancer] *********************************************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************************
ok: [lb01]

TASK [verify nginx service] *************************************************************************************************************************
 [WARNING]: Consider using the service module rather than running 'service'.  If you need to use command because service is insufficient you can add
'warn: false' to this command task or set 'command_warnings=False' in ansible.cfg to get rid of this message.

changed: [lb01]

TASK [verify nginx is listening on 80] **************************************************************************************************************
ok: [lb01]

PLAY [webserver] ************************************************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************************
ok: [app01]
ok: [app02]

TASK [verify apache2 service] ***********************************************************************************************************************
changed: [app01]
changed: [app02]

TASK [verify apache2 is listening on 80] ************************************************************************************************************
ok: [app01]
ok: [app02]

PLAY [database] *************************************************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************************
ok: [db01]

TASK [verify mysql service] *************************************************************************************************************************
changed: [db01]

TASK [verify mysql is listening on 3306] ************************************************************************************************************
ok: [db01]

PLAY [loadbalancer] *********************************************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************************
ok: [lb01]

TASK [verify backend index response] ****************************************************************************************************************
ok: [lb01] => (item=app01)
ok: [lb01] => (item=app02)

// 考えられるのはローカルMacとLBの
// /etc/hostsの名前解決先が内部IPと外部IPで異なる点

TASK [fail] *****************************************************************************************************************************************
 [WARNING]: conditional statements should not include jinja2 templating delimiters such as {{ }} or {% %}. Found: 'Hello, from sunny {{item.item}}!'
not in item.content

failed: [lb01] (item={'status': 200, 'content_length': '38', 'cookies': {}, 'date': 'Sun, 08 Sep 2019 04:36:43 GMT', 'url': 'http://app01', 'changed': False, 'vary': 'Accept-Encoding', 'server': 'Apache/2.4.25 (Debian)', 'content': 'Hello, from sunny ansible-template-3!\n', 'invocation': {'module_args': {'directory_mode': None, 'force': False, 'remote_src': None, 'status_code': [200], 'body_format': 'raw', 'owner': None, 'follow': False, 'client_key': None, 'group': None, 'use_proxy': True, 'headers': {}, 'unsafe_writes': None, 'serole': None, 'content': None, 'setype': None, 'follow_redirects': 'safe', 'return_content': True, 'client_cert': None, 'body': None, 'timeout': 30, 'src': None, 'dest': None, 'selevel': None, 'force_basic_auth': False, 'removes': None, 'http_agent': 'ansible-httpget', 'regexp': None, 'url_password': None, 'url': 'http://app01', 'validate_certs': True, 'seuser': None, 'method': 'GET', 'creates': None, 'unix_socket': None, 'delimiter': None, 'mode': None, 'url_username': None, 'attributes': None, 'backup': None}}, 'connection': 'close', 'content_type': 'text/html; charset=utf-8', 'msg': 'OK (38 bytes)', 'redirected': False, 'elapsed': 0, 'cookies_string': '', 'failed': False, 'item': 'app01', 'ansible_loop_var': 'item'}) => {"ansible_loop_var": "item", "changed": false, "item": {"ansible_loop_var": "item", "changed": false, "connection": "close", "content": "Hello, from sunny ansible-template-3!\n", "content_length": "38", "content_type": "text/html; charset=utf-8", "cookies": {}, "cookies_string": "", "date": "Sun, 08 Sep 2019 04:36:43 GMT", "elapsed": 0, "failed": false, "invocation": {"module_args": {"attributes": null, "backup": null, "body": null, "body_format": "raw", "client_cert": null, "client_key": null, "content": null, "creates": null, "delimiter": null, "dest": null, "directory_mode": null, "follow": false, "follow_redirects": "safe", "force": false, "force_basic_auth": false, "group": null, "headers": {}, "http_agent": "ansible-httpget", "method": "GET", "mode": null, "owner": null, "regexp": null, "remote_src": null, "removes": null, "return_content": true, "selevel": null, "serole": null, "setype": null, "seuser": null, "src": null, "status_code": [200], "timeout": 30, "unix_socket": null, "unsafe_writes": null, "url": "http://app01", "url_password": null, "url_username": null, "use_proxy": true, "validate_certs": true}}, "item": "app01", "msg": "OK (38 bytes)", "redirected": false, "server": "Apache/2.4.25 (Debian)", "status": 200, "url": "http://app01", "vary": "Accept-Encoding"}, "msg": "index failed to return content"}
failed: [lb01] (item={'status': 200, 'content_length': '24', 'cookies': {}, 'date': 'Sun, 08 Sep 2019 04:36:44 GMT', 'url': 'http://app02', 'changed': False, 'vary': 'Accept-Encoding', 'server': 'Apache/2.4.25 (Debian)', 'content': 'Hello, from sunny test!\n', 'invocation': {'module_args': {'directory_mode': None, 'force': False, 'remote_src': None, 'status_code': [200], 'body_format': 'raw', 'owner': None, 'follow': False, 'client_key': None, 'group': None, 'use_proxy': True, 'headers': {}, 'unsafe_writes': None, 'serole': None, 'content': None, 'setype': None, 'follow_redirects': 'safe', 'return_content': True, 'client_cert': None, 'body': None, 'timeout': 30, 'src': None, 'dest': None, 'selevel': None, 'force_basic_auth': False, 'removes': None, 'http_agent': 'ansible-httpget', 'regexp': None, 'url_password': None, 'url': 'http://app02', 'validate_certs': True, 'seuser': None, 'method': 'GET', 'creates': None, 'unix_socket': None, 'delimiter': None, 'mode': None, 'url_username': None, 'attributes': None, 'backup': None}}, 'connection': 'close', 'content_type': 'text/html; charset=utf-8', 'msg': 'OK (24 bytes)', 'redirected': False, 'elapsed': 0, 'cookies_string': '', 'failed': False, 'item': 'app02', 'ansible_loop_var': 'item'}) => {"ansible_loop_var": "item", "changed": false, "item": {"ansible_loop_var": "item", "changed": false, "connection": "close", "content": "Hello, from sunny test!\n", "content_length": "24", "content_type": "text/html; charset=utf-8", "cookies": {}, "cookies_string": "", "date": "Sun, 08 Sep 2019 04:36:44 GMT", "elapsed": 0, "failed": false, "invocation": {"module_args": {"attributes": null, "backup": null, "body": null, "body_format": "raw", "client_cert": null, "client_key": null, "content": null, "creates": null, "delimiter": null, "dest": null, "directory_mode": null, "follow": false, "follow_redirects": "safe", "force": false, "force_basic_auth": false, "group": null, "headers": {}, "http_agent": "ansible-httpget", "method": "GET", "mode": null, "owner": null, "regexp": null, "remote_src": null, "removes": null, "return_content": true, "selevel": null, "serole": null, "setype": null, "seuser": null, "src": null, "status_code": [200], "timeout": 30, "unix_socket": null, "unsafe_writes": null, "url": "http://app02", "url_password": null, "url_username": null, "use_proxy": true, "validate_certs": true}}, "item": "app02", "msg": "OK (24 bytes)", "redirected": false, "server": "Apache/2.4.25 (Debian)", "status": 200, "url": "http://app02", "vary": "Accept-Encoding"}, "msg": "index failed to return content"}

PLAY RECAP ******************************************************************************************************************************************
app01                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
app02                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
db01                       : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
lb01                       : ok=5    changed=1    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

```
// レスポンスは帰っているのでなぜfail扱いになるのか謎
"Hello, from sunny ansible-template-3!\n"
"Hello, from sunny test!\n"
```