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
