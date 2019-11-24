# molecule,ansible,dockerを使用したplaybookのテスト

- テストを通過し、冪等性を担保したplaybookを作成する

---
## molecule,ansible,docker環境の導入まで

---
### 必要なツールのインストール

- brewの確認

```
$ brew -v                           [/Users/kiyotatakeshi/Desktop/GitHub/Ansible]
Homebrew 2.1.12
Homebrew/homebrew-core (git revision a1e2; last commit 2019-10-04)
```

---
- pipのインストール
    - pythonのパッケージ管理システム

```
$ curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
$ python get-pip.py --user

# bashの場合は ~/bash_profile 等に書き込む
$ echo 'export PATH="$HOME/Library/Python/2.7/bin:$PATH"' >> ~/.zshrc

$ source ~/.zshrc        [/Users/kiyotatakeshi]

# 一応、アップグレード
$ pip install -U pip            [/Users/kiyotatakeshi]

```

---
- pyenvのインストール
    - バージョンを指定してpythonをインストール、適用できるツール
    - 特定のディレクトリ、システム全体のpythonのバージョンを指定可能

```
$ brew install pyenv

$ pyenv --version               [/Users/kiyotatakeshi]
pyenv 1.2.12

# bashの場合は ~/bash_profile 等に書き込む
$ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
$ echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
$ echo 'eval "$(pyenv init -)"' >> ~/.zshrc
$ source ~/.zshrc

```

---
- python3.7.3のインストール

```
$ pyenv install --list

$ pyenv install 3.7.3
Installed Python-3.7.3 to /Users/kiyotatakeshi/.pyenv/versions/3.7.3

# システム全体のバージョンを以下に設定
$ pyenv global 3.7.3

$ python --version
Python 3.7.3

$ which python
/Users/kiyotatakeshi/.pyenv/shims/python
```

---
- pipenvのインストール
    - ディレクトリごとに、pythonのバージョンとパッケージをまとめて切り替えるためのツール
    - Pipefileを使用して管理する

```
$ pip install pipenv --user

$ pipenv --version              [/Users/kiyotatakeshi]
pipenv, version 2018.11.26

# brew を使う場合
# $ brew install pipenv
```

---
### moleculeの使用

- ansibleのテストフレームワーク
    - 仮想環境の準備から、playbookの実行、環境の破棄までを行う

- pipenvで作成した仮想環境内で、moleculeのインストール
    - 適当なディレクトリを作成し、テスト環境を作ってみる

```
$ pwd
/tmp/molecule_test

$ pipenv install molecule

# まだ仮想環境に入っていないので、moleculeはインストールされていない
$ molecule --version
zsh: command not found: molecule
```

- Pipfileが作成される

```
$ ls
Pipfile       Pipfile.lock

$ cat Pipfile
[[source]]
name = "pypi"
url = "https://pypi.org/simple"
verify_ssl = true

[dev-packages]

[packages]
molecule = "*"

[requires]
python_version = "3.7"
```

- 仮想環境に入る

```
$ pipenv install
$ pipenv shell

# 環境を更新するときは
# $ pipenv update

# 仮想環境であることが、プロンプトに表示される
# moleculeがインストールされている
(molecule_test) $ molecule --version
molecule, version 2.22

```

- moleculeでテストシナリオを作成

```
(molecule_test) $ molecule init role -r test_role
--> Initializing new role test_role...
Initialized role in /private/tmp/molecule_test/test_role successfully.

# test_role の内容を書き換えて使用する
(molecule_test) $ tree -L 2
.
├── Pipfile
├── Pipfile.lock
└── test_role
    ├── README.md
    ├── defaults
    ├── handlers
    ├── meta
    ├── molecule
    ├── tasks
    └── vars

```
---
## unit_test_sample の作成

```
$ tree -L 2 unit_test_sample
unit_test_sample
├── Pipfile
├── Pipfile.lock
├── defaults
│   └── main.yml
├── molecule
│   ├── docker
│   ├── shared
│   └── vagrant
├── tasks
│   ├── configure_mysql.yml
│   ├── main.yml
│   ├── secure_installation.yml
│   └── setup_mysql.yml
├── templates
│   └── my.cnf.j2
└── vars
    └── main.yml

```

- 仮想環境の作成

```
$ pipenv install
$ pipenv shell
Launching subshell in virtual environment…

# Pipfileに記載したものがインストールされている
(unit_test_sample) $ molecule --version
molecule, version 2.22

(unit_test_sample) $ ansible --version
ansible 2.9.1

(unit_test_sample) $ docker --version
Docker version 19.03.4, build 9013bf5

(unit_test_sample) $ vagrant --version
Vagrant 2.2.6

```

- docker内でplaybookの実行

```
# pipenv run で scripts が実行される
(unit_test_sample) $ tail -n 15 Pipfile
[requires]
python_version = "3.7"

[scripts]
test = "molecule --base-config ./molecule/docker/molecule.yml --env-file ./molecule/shared/.env.yml test --scenario-name docker"
converge = "molecule --base-config ./molecule/docker/molecule.yml --env-file ./molecule/shared/.env.yml converge --scenario-name docker"
login = "molecule --base-config ./molecule/docker/molecule.yml --env-file ./molecule/shared/.env.yml login --scenario-name docker"
destroy = "molecule --base-config ./molecule/docker/molecule.yml --env-file ./molecule/shared/.env.yml destroy --scenario-name docker"
syntax = "molecule --base-config ./molecule/docker/molecule.yml --env-file ./molecule/shared/.env.yml --debug syntax --scenario-name docker"

test-vagrant = "molecule --base-config ./molecule/docker/molecule.yml --env-file ./molecule/shared/.env.yml test --scenario-name vagrant"
converge-vagrant = "molecule --base-config ./molecule/docker/molecule.yml --env-file ./molecule/shared/.env.yml converge --scenario-name vagrant"
login-vagrant = "molecule --base-config ./molecule/docker/molecule.yml --env-file ./molecule/shared/.env.yml login --scenario-name vagrant"
destroy-vagrant = "molecule --base-config ./molecule/docker/molecule.yml --env-file ./molecule/shared/.env.yml destroy --scenario-name vagrant"
syntax-vagrant = "molecule --base-config ./molecule/docker/molecule.yml --env-file ./molecule/shared/.env.yml --debug syntax --scenario-name vagrant"

# docker内でテストの実行
# convergeはテスト作成したインスタンスをdestroyしない
(unit_test_sample) $ pipenv run converge

--> Test matrix

└── docker
    ├── dependency
    ├── create
    ├── prepare
    └── converge

# docker内でplaybookが正常に実行された
   PLAY RECAP *********************************************************************
    instance                   : ok=14   changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

- テストを実行したdockerにログイン

```
(unit_test_sample) $ pipenv run login
--> Validating schema /Users/kiyotatakeshi/Desktop/GitHub/Ansible/molecule/unit_test_sample/molecule/docker/molecule.yml.
Validation completed successfully.
--> Validating schema /Users/kiyotatakeshi/Desktop/GitHub/Ansible/molecule/unit_test_sample/molecule/vagrant/molecule.yml.
Validation completped successfully.

[root@instance /]#

# mysqlをインストールできている
[root@instance /]# mysql --version
mysql  Ver 14.14 Distrib 5.7.28, for Linux (x86_64) using  EditLine wrapper

[root@instance /]# mysql -uroot
mysql>
```

### gossの導入

- [golangで書かれ、yamlで記述できるテストツール](https://github.com/aelsabbahy/goss)

- インストール

```
[root@instance /]# curl -L https://github.com/aelsabbahy/goss/releases/download/v0.3.7/goss-linux-amd64 -o /usr/local/bin/goss

[root@instance /]# ls -l /usr/local/bin/goss
-rw-r--r-- 1 root root 9825536 Nov 24 01:20 /usr/local/bin/goss

[root@instance /]# chmod +rx /usr/local/bin/goss

[root@instance /]# ls -l /usr/local/bin/goss
-rwxr-xr-x 1 root root 9825536 Nov 24 01:20 /usr/local/bin/goss

[root@instance /]# goss --version
goss version v0.3.7

```

- [テストファイルを作成](https://github.com/aelsabbahy/goss/blob/master/docs/manual.md)
    - 基本的には autoadd で自動作成される
    
```
# サービスが起動していることを確認
[root@instance /]# systemctl is-active mysqld
active

[root@instance /]# goss autoadd mysqld
Adding Process to './goss.yaml': # カレントディレクトリにテスト内容を記載したファイルが自動作成される

[root@instance /]# goss autoadd /root/.my.cnf

[root@instance /]# goss autoadd mysql

# インストールできているか確認したいパッケージを表示
[root@instance /]# rpm -qa | grep "mysql"
mysql57-community-release-el7-11.noarch
mysql-community-common-5.7.28-1.el7.x86_64
mysql-community-libs-compat-5.7.28-1.el7.x86_64
mysql-community-server-5.7.28-1.el7.x86_64
mysql-community-libs-5.7.28-1.el7.x86_64
mysql-community-client-5.7.28-1.el7.x86_64

[root@instance /]# goss autoadd $(rpm -qa | grep "mysql")

# 自動作成されたファイルを編集
[root@instance /]# vi goss.yaml

# あとはこれに command で細かなテスト項目を追加する
[root@instance /]# cat goss.yaml
file:
  /root/.my.cnf:
    exists: true
    mode: "0600"
    owner: root
    group: root
    filetype: file
    contains: []

user:
  mysql:
    exists: true
    groups:
    - mysql
    home: /var/lib/mysql
    shell: /bin/false

group:
  mysql:
    exists: true
    gid: 27

package:
  mysql-community-client-5.7.28-1.el7.x86_64:
    installed: true
    versions:
    - 5.7.28
  mysql-community-common-5.7.28-1.el7.x86_64:
    installed: true
    versions:
    - 5.7.28
  mysql-community-server-5.7.28-1.el7.x86_64:
    installed: true
    versions:
    - 5.7.28

service:
  mysqld:
    enabled: true
    running: true

process:
  mysqld:
    running: true
    
```

- テスト環境(docker)の削除

```
(unit_test_sample) $ pipenv run destroy
```

---
### unit_test_sample でテストを実行

- 冪等性のチェックとテストフレームワークまで走らせる
    - これを通過したplaybookは安心して使用できる

```
(unit_test_sample) $ pipenv run test

--> Test matrix

└── docker
    ├── lint
    ├── dependency
    ├── cleanup
    ├── destroy
    ├── syntax
    ├── create
    ├── prepare
    ├── converge
    ├── idempotence
    ├── side_effect
    ├── verify
    ├── cleanup
    └── destroy

# docker内でplaybookが正常に実行された
    PLAY RECAP *********************************************************************
    instance                   : ok=14   changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

# 冪等性のチェック(再度playbookを実行して、すべてOKだと通過)
--> Action: 'idempotence'
Idempotence completed successfully. # changed がでるとエラーになる

# テストフレームワークの実行(デファルトはtestinfra、今回はgoss)
--> Action: 'verify'
--> Executing Goss tests found in /Users/kiyotatakeshi/Desktop/GitHub/Ansible/molecule/unit_test_sample/molecule/docker/../shared/tests/...

# こちらも通過
   PLAY RECAP *********************************************************************
    instance                   : ok=6    changed=4    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

Verifier completed successfully.

# テストに使用したdockerを削除する
--> Scenario: 'docker'
--> Action: 'cleanup'
Skipping, cleanup playbook not configured.
--> Scenario: 'docker'
--> Action: 'destroy'

    PLAY [Destroy] *****************************************************************

    TASK [Destroy molecule instance(s)] ********************************************
    changed: [localhost] => (item=instance)

    TASK [Wait for instance(s) deletion to complete] *******************************
    FAILED - RETRYING: Wait for instance(s) deletion to complete (300 retries left).
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Delete docker network(s)] ************************************************

    PLAY RECAP *********************************************************************
    localhost                  : ok=2    changed=2    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

--> Pruning extra files from scenario ephemeral directory

```

---
## integration_test_sample の作成
