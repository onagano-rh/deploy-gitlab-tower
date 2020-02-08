<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [How to Use](#how-to-use)
    - [Inventory](#inventory)
    - [Ansible Vaultによる秘密情報の暗号化について](#ansible-vaultによる秘密情報の暗号化について)
    - [各Playbookの実行方法](#各playbookの実行方法)
        - [GitLabとTowerのインストール (site.yml, deploy_gitlab.yml, deploy_tower.yml)](#gitlabとtowerのインストール-siteyml-deploygitlabyml-deploytoweryml)
        - [Runnerのインストール (deploy_gitlab-runner.yml)](#runnerのインストール-deploygitlab-runneryml)
        - [GitLabのバックアップとリストア (backup_gitlab.yml, restore_gitlab.yml)](#gitlabのバックアップとリストア-backupgitlabyml-restoregitlabyml)
        - [Towerのバックアップとリストア (roles/tower)](#towerのバックアップとリストア-rolestower)
- [Old Notes](#old-notes)
    - [インベントリの設定](#インベントリの設定)
    - [ansible.cfgの設定](#ansiblecfgの設定)
    - [Ansible Vaultのパスワードファイルの作成と暗号化文字列の更新](#ansible-vaultのパスワードファイルの作成と暗号化文字列の更新)
    - [GitLabとGitLab Runner間の自己署名証明書の問題の回避策](#gitlabとgitlab-runner間の自己署名証明書の問題の回避策)
    - [実行](#実行)
    - [GitLabの初回アクセス時のrootパスワードの変更](#gitlabの初回アクセス時のrootパスワードの変更)
    - [Towerでの自己署名証明書の許可](#towerでの自己署名証明書の許可)
    - [GitLab Runnerのインストール](#gitlab-runnerのインストール)

<!-- markdown-toc end -->


How to Use
=============

Inventory
-----------

| Group         | 推奨OS   | 備考 |
| -----         | ------   | ---- |
| gitlab        | CentOS 7 | [geerlingguy.gitlab](https://galaxy.ansible.com/geerlingguy/gitlab) を使用。 |
| gitlab_runner | CentOS 7 | [riemers.gitlab-runner](https://galaxy.ansible.com/riemers/gitlab-runner) を使用。 |
| tower         | RHEL 8   | [公式インストーラ](https://releases.ansible.com/ansible-tower/setup/) と適切なサブスクリプションを使用。 |

各グループにホストを1台ずつ用意すること。
GitLab Runnerは複数台でも可だが、他はHA構成に対応したPlaybookにはなっていない。

GitLabに関しては、公式のインストーラがバージョン7系にしか対応していないためCentOS 7にした。

以下、Ansible TowerをTower、GitLab RunnerをRunnerと単に呼ぶことにする。

Ansible Galaxyの各Roleは、Towerでの実行時は roles/requirements.yml の情報を用いて
自動取得することもできるが、ここではバージョンを固定する意味も含めてリポジトリ内に含めた。


Ansible Vaultによる秘密情報の暗号化について
----------------------------------------------

特にTowerのインストールでは、サブスクリプションのパスワードや
ライセンスのJSONオブジェクトなどの変数を暗号化して保存している。
これらは各ユーザが`ansible-vault`コマンドを使って自分のものを生成し、
roles/tower/defaults/main.yml を書き換える必要がある。

また、そうした変数を使うPlaybookの実行時には`--vault-password-file`
オプションの指定も必要になる。

```shell
echo -n '<Your_Password>' > /tmp/mypassword.txt
echo -n '<Variable_Value>' | ansible-vault encrypt_string \
  --vault-password-file /tmp/my-password.txt --stdin-name <Variable_Key>
vi roles/tower/defaults/main.yml
ansible-playbook --vault-password-file /tmp/my-password.txt your_playbook.yml
```


各Playbookの実行方法
----------------------

### GitLabとTowerのインストール (site.yml, deploy_gitlab.yml, deploy_tower.yml)

site.yml でGitLabとTowerのインストールを行う。
これは中で deploy_gitlab.yml と deploy_tower.yml を呼び出しているだけである。

```shell
ansible-playbook --vault-password-file /tmp/my-password.txt site.yml
```

| System | 管理者ユーザ | そのパスワード                                                   |
| ------ | ------------ | -----------                                                      |
| GitLab | root         | 5iveL!fe（初回アクセス時に変更する）                             |
| Tower  | admin        | （roles/tower/defaults/main.yml の "admin_password" 変数で設定） |

TowerがGitLabのプロジェクトをクローンできるように、
自己署名証明書の検証を行わない下記の設定を "SETTINGS / JOBS / EXTRA ENVIRONMENT VARIABLES"
に行う。

```
{
 "GIT_SSL_NO_VERIFY": "True"
}
```

また、roles/requirements.yml を用いたRoleやCollectionの自動ダウンロードを
無効にする設定も同じ画面でできる。



### Runnerのインストール (deploy_gitlab-runner.yml)

Runnerについては gitlab-runner.yml を後で別に実行してインストールする。
事前にGitLabの "Admin Area > Runners" から登録用のトークンを取得しておき、
Extra Varsで指定する。

```shell
ansible-playbook deploy_gitlab-runner.yml \
  -e 'gitlab_runner_coordinator_url=https://gitlab.example.com/' \
  -e 'gitlab_runner_registration_token=<Registration_Token>'
```


 Install GitLab Runner
Specify the following URL during the Runner setup: https://gitlab.example.com/
Use the following registration token during setup: -NLCEjjgCK8_JPxQ2oRx
Start the Runner! 


### GitLabのバックアップとリストア (backup_gitlab.yml, restore_gitlab.yml)

変数"local_dest"を指定した場合は、コントロールノードにバックアップファイルをコピーする。
この変数が無い場合は、リモートホスト側にバックアップファイルを残すだけになる。

```shell
ansible-playbook backup_gitlab.yml -e local_dest=.
```

GitLabではデータとセキュリティ関連の設定とでバックアップ対象を2つに分けており、
ファイルも2つできる。

```
$ ls
gitlab.example.com_1581179648_2020_02_08_12.7.5_gitlab_backup.tar
gitlab.example.com_gitlab_config_1581179650_2020_02_08.tar
```

これらを用いてリストアを行える。

```
ansible-playbook restore_gitlab.yml \
  -e gitlab_backup_data=./gitlab.example.com_1581179648_2020_02_08_12.7.5_gitlab_backup.tar \
  -e gitlab_backup_config=./gitlab.example.com_gitlab_config_1581179650_2020_02_08.tar
```


### Towerのバックアップとリストア (roles/tower)

TowerではRoleを自作してることもあり、バックアップとリストアのタスクは
Role内に記述しており、実行時にタグを指定することで使い分ける。

```
ansible-playbook deploy_tower.yml -t tower_backup -e local_dest=.
```

Towerではバックアップファイルは1つであり、それを変数"tower_backup_file"に
指定してタグ"tower_restore"を指定してリストアを行う。

```
ansible-playbook deploy_tower.yml -t tower_restore \
  -e tower_backup_file=tower.example.com_tower-backup-2020-02-08-11:40:48.tar.gz
```




Old Notes
=========


インベントリの設定
------------

GitLab用にCentOS 7サーバーが1台、
Ansible Tower（以下単に"Tower"と呼ぶ）用にRHEL 8サーバーが1台、
それぞれ必要です。

`hosts.example`を各自コピーし編集して使います。
カレントディレクトリに`./hosts`としてコピーし
同ディレクトリ内で`ansible-playbook`コマンドを実行するのであれば、
同ディレクトリのansible.cfgが読み込まれてデフォルトのインベントリとみなされ
`-i`オプションの指定が省略できます。

他に、`~/.ansible/hosts`にコピーすれば、
実行ディレクトリに拘わらず`-i`オプションを省略できます。


ansible.cfgの設定
--------------

`ansible-playbook`コマンドを実行するディレクトリに`ansible.cfg`があれば
それが読み込まれます。

カレントディレクトリに同ファイルがなければ`~/.ansible.cfg`が、
それもなければ`/etc/ansible/ansible.cfg`が使われます。


Ansible Vaultのパスワードファイルの作成と暗号化文字列の更新
--------------

現在、以下の情報をAnsible Vaultを使って暗号化して保存しています。

- RHELのサブスクリプションのパスワード
- Towerインストール時の各パスワード3つ（Tower, PostgreSQL, RabbitMQ）
- TowerのライセンスのJSONオブジェクト

これにより秘密を知られることなくGitリポジトリで保存・公開できます。

Ansible Vaultでの暗号化や復号化には別のパスワードが必要で、
通常は必要になるたびに一時ファイルに保存してそれを使用します。
(Towerでの実行時にはTower内に保存できる。)

以下の例では`/tmp/my-password.txt`にパスワードが保存されているとします。
ファイル中の最後に改行が含まれないよう、以下のようなコマンドで作成します。

```
echo -n '<your_password>' > /tmp/my-password.txt
```

ただし、この場合はシェルの履歴にパスワードが残る可能性に注意してください。

別の方法として、`vi`で編集し最後に付いてしまう改行を`tr -d '\n'`のフィルターを
用いて実行時に削除するというのがあります。

現在ファイルに保存されている暗号化文字列は私自身のものですので、
別の使用者は自身で暗号化文字列を更新して保存する必要があります。

ファイル`/tmp/my-value.txt`に暗号化したい元の文字列があるとして、
以下のコマンドで標準出力に暗号化文字列を表示できますので、
これをコピペします。


```
cat /tmp/my-value.txt | tr -d '\n' | \
  ansible-vault encrypt_string --vault-password-file /tmp/my-password.txt \
  --stdin-name subscription_password
```

最後の`subscription_passowrd`は変数名で、適宜変更して下さい。

実際に変更が必要なファイルと変数のリストは以下の通りです。
"*"を付けたものは暗号化が必要です。

- roles/common/vars/RedHat_8.yml
  - subscription_username
  - subscription_password *
  - subscription_pool_id
- roles/tower/defaults/main.yml
  - admin_password *
  - pg_password *
  - rabbitmq_password *
  - tower_license *

最後の`tower_lisense`にはTowerのライセンスのJSONオブジェクトを格納します。
このとき、下記のように`"eula_accepted": true`が含まれていないと
ライセンス登録時にエラーになりますので、無い場合は追加してください。


```
{
  ...
  "license_key": "...",
  "eula_accepted": true,
  ...
}
```

GitLabとGitLab Runner間の自己署名証明書の問題の回避策
----------------

後でGitLab Runnerをインストールするときに、
GitLab Runnerが自分をGitLabの証明書を正しく検証できないとエラーになります。
これを回避するために、GitLabの自己署名証明書をGitLab Runner側にコピーする
作業が、GitLab Runnerのインストール時に行われます。
それに備えて、GitLabの証明書のサブジェクトが正しいものになるように
deploy_gitlab.ymlで下記の変数を定義しています。

    gitlab_external_url: "https://{{ inventory_hostname }}/"
    gitlab_self_signed_cert_subj: "/C=JP/ST=Tokyo/L=Shibuya/O=Consulting/CN={{ inventory_hostname }}"

これらの変数を未定義のままGitLabをインストールした場合は、
再インストールするか、下記のタスクで行っているように`openssl`コマンドで再作成し配置し直します。

https://github.com/geerlingguy/ansible-role-gitlab/blob/2.5.2/tasks/main.yml#L62


実行
----------------

GitLabにはAnsible Galaxyにある
[geerlingguy](https://galaxy.ansible.com/geerlingguy/gitlab)
を再利用しており、下記のコマンドであらかじめ取得したものをリポジトリにも収めています。

```
ansible-galaxy role install -r roles/requirements.yml -p roles
```

その後、以下のコマンドで全体をデプロイします。

```
ansible-playbook [-i /your/hosts] --vault-password-file /tmp/my-password.txt site.yml
```

`-i`オプションによるインベントリの指定は前述のように省略可能です。

`site.yml`では2つのPlaybookをインポートしているだけで、両者は独立しているので、
それぞれを別のターミナルで並列に実行することもできます。


```
# ターミナル1
ansible-playbook deploy_gitlab.yml
# ターミナル2
ansible-playbook --vault-password-file /tmp/mypassword.txt deploy_tower.yml

```

GitLabの場合は、OSがCentOSならば暗号化文字列は使わないため
パスワードファイルの指定は要りません。

何度か繰り返し実行する場合は、`-t`や`--skip-tags`オプションを用いて
タグのついた特定部分のみを実行するのが便利です。



GitLabの初回アクセス時のrootパスワードの変更
----------------

GitLabの管理者のパスワードはファイルで指定するのではなく、
`root/5iveL!fe`で固定で、ブラウザでの初回アクセス時に任意のものに変更します。

なお、Towerの管理ユーザは`admin`、
パスワードは前述の`admin_password`変数でしていしたものです。


Towerでの自己署名証明書の許可
-----------------

GitLabが自己署名証明書を用いているため、そのままではTowerがそこから
リポジトリをクローンするときにエラーになってしまいます。
これを回避するために、"SETTINGS / JOBS / EXTRA ENVIRONMENT VARIABLES"
に以下の設定を行います。

```
{
 "GIT_SSL_NO_VERIFY": "True"
}
```

GitLab Runnerのインストール
------------------

GitLabでCI/CDを行うには、GitLab本体とは別にGitLab Runnerをインストールします。
GitLab Runnerではビルド中に危険な操作（`rm -rf /`など）も行い得るため、
GitLab本体とは別のサーバにインストールすることが強く推奨されています。

ここではGalaxy Hubにあるriemers.ansible-gitlab-runnerを使います。

https://github.com/riemers/ansible-gitlab-runner

GitLabの画面上でRegistration Tokenを取得する必要があるため、
一旦GitLabのインストールを終えて、その後GitLab Runnerをインストールします。

GitLabにrootでログインし、下記のURLにアクセスしてURLとRegistration Tokenをメモします。

https://gitlab.example.com/admin/runners

その後以下のようにURLとRegistration Tokenを指定してdeploy_gitlab-runner.ymlを実行します。

```
ansible-playbook deploy_gitlab-runner.yml \
  -e 'gitlab_runner_coordinator_url=<GitLabのURL>'
  -e 'gitlab_runner_registration_token=<Registration_Token>'
```
